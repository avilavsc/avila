---
title: 'Idempotência: o teu sistema aguenta a segunda chamada?'
date: 2026-11-24
draft: false
tags: ['idempotencia', 'fundamentos']
featureimage: 'featured.png'
showHero: true
heroStyle: 'big'
---

Olha, é muito importante que tu faça as tuas regras de negócio idempotentes. Pelo menos as críticas.

Eu vejo muitos programadores perderem um tempão com bugs que poderiam ser evitados com o uso de chamadas que têm o poder da idempotência.

O nome assusta um pouco, mas o conceito é bem comum para nós devs (ou deveria ser).

Idempotência é rodar um trecho de código 1 ou N vezes e obter o mesmo resultado. Sem quebrar o estado da aplicação.

Neste texto eu abordo o problema de não considerar a idempotência de uma aplicação, uma forma prática de lidar com isso e a postura do programador que estuda fundamentos e não aceita apenas o caminho feliz.

## Operações duplicadas

O teu sistema consegue lidar com operações duplicadas? Eu te pergunto isso porque é muito comum eu ver uma importação de arquivo `CSV` que não pode subir duas vezes. Isso pra mim é estranho, mas é bem comum.

Por exemplo, uma planilha de clientes (bem simples). Se tu subir essa planilha com um script que tu escreveu na mão, ela vai e cria os clientes. Ok. Tudo certo.

Mas em algum outro momento outra pessoa vai e roda o script sem querer utilizando a mesma planilha. E aí, o que acontece?

Os clientes são duplicados com id diferente. Se a chave primária da tabela `PK` é um id gerado automaticamente, é isso. Este é um cenário bem ruim.

Uma outra opção é que a chave primária seja um CPF, por exemplo. A situação é um pouco diferente. Planilhas iguais --> erro de unicidade no primeiro cliente --> não roda mais nada. Nesse caso o estado da aplicação fica preservado. Talvez ainda apareça um erro feio para o usuário (`SQLException`), mas se for um script executado pela TI, até passa.

Mas nesse caso temos um problema diferente: se as planilhas vêm de várias fontes, os dados podem estar misturados e repetidos em diversas planilhas. A situação agora é que o script roda, começa a importar os clientes novos até achar o primeiro repetido --> estoura erro de SQL --> para de importar o resto. Importamos uma parte da planilha, não sabemos quanto, e ao rodar de novo já dá erro no primeiro, porque o CPF tem que ser único.

Não vou entrar agora neste escopo, mas estes exemplos lidam apenas com uma aplicação. Em ambientes distribuídos a coisa fica ainda pior. Esses erros podem ser silenciosos.

Um serviço pode mandar uma mensagem para outro e falhar de forma invisível aos olhos do cliente. Se o usuário simplesmente tentar de novo *retry* sem proteção no servidor, tu corre o risco de fazer uma cobrança duas vezes, duplicar agendamentos e outras coisas que fazem as pessoas te ligarem às 23h de um domingo.

## Fundamentos

Pra começar, quero dizer que, como tudo na computação, alguém já passou por isso, já resolveu e já está documentado por aí.

Quando tu estiver quebrando a cabeça com coisas comuns (não estou dizendo simples), coisas que são padrão de mercado, como uma API de pagamento ou uma importação de um arquivo `CSV`, liga o teu alerta, pois algo pode estar errado.

E isso pode ser alguma coisa fundamental, como idempotência, *ACID*, concorrência, normalização. Essas coisas que a gente lê e passa batido até ter que usar de verdade. Essas coisas que chamamos de fundamentos e que ninguém gosta de postar no LinkedIn tanto quanto uma *SPA* feita em React (nada contra, foi só um exemplo).

Coisas que não vendem curso, que não te prometem vaga na gringa. Enfim, estudem fundamentos.

Nos meus estudos para concurso, muitas vezes eu tive que revisitar profundamente esses temas. E isso me gerou o hábito de procurar o nome das coisas para resolver problemas. Problemas conhecidos geram soluções conhecidas, esse é o negócio. E se a solução for realmente boa, vira literatura.

E bingo, é por isso que ainda aprendemos C na faculdade lá no início, e não Node. Porque os fundamentos da linguagem `C` e *C-like* são perenes, são estado da arte. E não vão sumir com o lançamento do próximo framework *JS*.

## Como implementar idempotência?

Existem muitas formas, e escolhi esta pela proximidade com os exemplos que dei. Uma das maneiras é usar uma chave composta que não se repita no contexto: idempotência por chaves naturais.

Em vez de gerar uma chave artificial só para controlar a duplicidade (um `UUID` enviado a cada requisição, por exemplo), a chave natural usa informações que já identificam a ação por natureza.

Imagina um pagamento: "O pagamento do pedido 123 foi aprovado". Se essa chamada, por algum motivo, chega duas vezes, podemos liberar o produto duas vezes, por exemplo.

Podemos aprimorar essa chamada com mais algumas chaves que identifiquem a ação de maneira única:

Ex.: "O pagamento do pedido 123 foi aprovado" --> "O pagamento do pedido 123, do cliente xyz, pelo provedor Stripe, foi aprovado".

Agora podemos criar uma constraint `UNIQUE` no banco que não aceite duas linhas com a combinação "pedido 123" + "cliente xyz" + "provedor Stripe".

E para não cairmos naquele erro tudo ou nada do tipo SQL EXCEPTION, podemos adicionar uma cláusula `ON CONFLICT ... DO NOTHING` no INSERT (sintaxe do PostgreSQL; no MySQL o equivalente é `INSERT IGNORE`). Quando o banco percebe que aquela combinação já existe, simplesmente ignora o INSERT e segue para o próximo, sem travar o sistema, sem duplicar o registro e sem efeitos inesperados.

Só libera o produto se o INSERT realmente inseriu a linha (confere as linhas afetadas), de preferência na mesma transação.

Pode testar aí e ver como vai funcionar na grande maioria dos casos. Adapta para a realidade da tua aplicação. Escolhe as chaves que representem uma ação única e irrepetível, e a constraint de unicidade do banco já vai te dar esse superpoder.

## No código

Veja um exemplo em SQL (PostgreSQL) do que mencionei acima. Não quero que o código seja *fancy*, quero que seja claro.

```sql
CREATE TABLE pagamentos_processados (
	id SERIAL PRIMARY KEY,
	cliente_id VARCHAR(100) NOT NULL,
	provedor VARCHAR(50) NOT NULL,
	numero_pedido VARCHAR(100) NOT NULL,
	valor DECIMAL(10, 2) NOT NULL,
	criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

	-- Trava composta pelas 3 colunas
	CONSTRAINT uq_cliente_provedor_pedido UNIQUE (cliente_id, provedor, numero_pedido)
);
```

Incluí **as três colunas (`cliente_id`, `provedor` e `numero_pedido`) dentro da mesma constraint `UNIQUE`**.

Dessa forma, o banco só bloqueia a inserção se os **três valores forem idênticos** ao mesmo tempo. Se o mesmo cliente fizer outro pedido no mesmo provedor, o banco aceita normalmente.

E o INSERT que aproveita essa trava:

```sql
INSERT INTO pagamentos_processados (cliente_id, provedor, numero_pedido, valor)
VALUES ('xyz', 'stripe', '123', 99.90)
ON CONFLICT (cliente_id, provedor, numero_pedido) DO NOTHING;
```

Pode rodar esse INSERT uma ou dez vezes: vai existir uma única linha. Isso é idempotência.

## Não espere o caminho feliz

Eu gostaria que, a partir de agora, tu começasse a desconfiar do caminho feliz. Nem sempre as coisas acontecem como o planejado, mas nós desenvolvedores precisamos prever, prevenir e mitigar os impactos quando as coisas saem do curso.

Não tem uma única fórmula. No meu caso, eu costumo ligar o modo paranoia quando é um requisito de negócio muito crítico. Tentar imaginar o que o usuário muito leigo e o usuário muito pro podem tentar fazer me ajuda bastante.

Além disso, estudar. Estudar é essencial para nós, profissionais de tecnologia. O volume de conhecimento aumenta constantemente, e as novidades chegam cada vez mais rápido. É preciso gostar de estudar, é preciso se orgulhar do trabalho que faz e do código que escreve, assim como ter o apetite de revisar aquele código que não está bem escrito ou que não entendemos tão bem.

Idempotência é um desses fundamentos: simples de entender, fácil de esquecer e caro de ignorar. Desconfia do caminho feliz e garante que o teu sistema aguente uma segunda ou terceira chamada sem medo.


Me segue no [LinkedIn](https://www.linkedin.com/in/avila-vasconcelos/) pra gente trocar ideia por lá.