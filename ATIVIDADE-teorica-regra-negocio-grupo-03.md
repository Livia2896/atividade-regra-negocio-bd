# Atividade Teórica: Regra de Negócio no BD versus na Aplicação

**Aluno(s):** Andreia Letícia Lemos, Layla Tais Sousa, Lívia Moreira, Maiany Melo, Sarha Sthefanny Araripe Silva, Yasmin de Lima
**Turma:** Banco de Dados 2026
**Data:** 19/08/2026
**Repositório Git:** https://github.com/Livia2896/atividade-regra-negocio-bd

## Resumo Executivo

## 1. Desenvolvimento Teórico

### 1.1 O que é regra de negócio?

### 1.2 Regras no banco de dados

Pra manter a coerência com o resto do trabalho, vamos usar o mesmo exemplo: uma loja de acessórios de tecnologia (teclado, fone sem fio, microfone de lapela), onde a regra que mais nos interessa é "não pode vender mais do que tem em estoque, mesmo se dois vendedores tentarem vender o mesmo item ao mesmo tempo".

**Constraints**
**CHECK**

Dá pra usar um *CHECK* simples pra garantir que o estoque nunca fique negativo, tipo *estoque >= 0* lá na tabela de produtos. Assim, mesmo que alguma venda seja processada errado, o banco não deixa o teclado ficar com "-2" no estoque.

**Vantagens:**
    1. Essa regra vale pra qualquer sistema que mexer no banco (site, app do vendedor, PDV da loja), não importa quem escreveu o código, ninguém consegue "burlar" a regra esquecendo de validar na aplicação.
    2. É simples de declarar e não exige escrever nenhuma lógica extra, só a condição direto na definição da coluna.
**Limitações:**
    1. Sozinha, ela não resolve o problema dos dois vendedores vendendo o último fone ao mesmo tempo, só barra o resultado final negativo, não coordena o que acontece durante a venda (isso só a gente resolve com transação, mais pra frente).
    2. Só valida regras simples sobre a própria linha/coluna, não dá pra usar CHECK, por exemplo, pra impedir que um cliente tenha dois pedidos em aberto ao mesmo tempo, porque isso depende de olhar outras linhas da tabela.
    
**FOREIGN KEY**

Aqui é o clássico: garantir que todo item de um pedido aponte pra um produto que realmente existe na tabela de produtos. Evita, por exemplo, um pedido "referenciando" um microfone de lapela que já foi removido do catálogo.

**Vantagens:**
    1. Evita esses pedidos "órfãos", que apontariam pra produtos inexistentes e quebrariam relatórios de venda.
    2. Pode propagar ações automáticas, tipo *ON DELETE CASCADE*, se um produto for descontinuado, sem precisar de código extra na aplicação pra isso.
**Limitações:**
    1. Se usarmos *CASCADE* sem pensar direito, dá pra acabar apagando o histórico de vendas inteiro só porque alguém excluiu um produto do catálogo.
    2. Toda *FK* impõe uma checagem extra em cada *INSERT/UPDATE/DELETE*, o que pode pesar um pouco a performance se a loja tiver um volume grande de vendas.
    
**UNIQUE**

Serve pra garantir, por exemplo, que o *SKU* (código) de cada produto seja único, não pode ter dois cadastros diferentes pro mesmo teclado.

**Vantagens:**
    1. Evita duplicidade sem precisar validar isso na aplicação, o banco recusa o cadastro duplicado automaticamente.
    2. De brinde, geralmente cria um índice, o que ajuda a busca por produto a ser mais rápida.
**Limitações:**
    1. Não pega duplicidade "disfarçada", tipo dois cadastros diferentes com nomes escritos de forma diferente pro mesmo produto (ex.: "Fone BT-100" e "fone bt 100").
    2. O comportamento com valores *NULL* varia de SGBD pra SGBD, alguns permitem vários *NULLs* numa coluna *UNIQUE*, o que pode gerar confusão se a equipe não souber disso.
    
**Triggers**

Dá pra criar uma trigger que, toda vez que o estoque de um produto mudar, registre automaticamente um log (quem vendeu, quando, quanto). Ou até uma trigger que atualiza o estoque assim que a venda é confirmada.

**Vantagens:**
    1. Funciona pra loja inteira, seja venda pelo site ou pelo PDV da loja física, sem depender de cada sistema lembrar de atualizar o estoque manualmente.
    2. É muito útil pra auditoria, dá pra manter um histórico de quem alterou o quê e quando, sem precisar que a aplicação implemente esse log.
**Limitações:**
    1. Fica meio "escondido", se um colega for ler só o código da aplicação, não vai entender de onde vem a atualização do estoque, o que dificulta a manutenção.
    2. Pode complicar de debugar se uma trigger disparar outra (efeito cascata), e cada banco tem sua própria sintaxe, então não dá pra simplesmente copiar de um SGBD pro outro.
    
**Stored Procedures**

Dava pra criar uma procedure tipo realizar_venda(produto_id, quantidade) que já faz tudo: confere se tem estoque, desconta a quantidade vendida e registra o pedido, tudo numa chamada só.

**Vantagens:**
    1. Reduz a ida e volta entre aplicação e banco (várias operações numa chamada só), o que ajuda na performance.
    2. A mesma lógica de venda pode ser reaproveitada tanto pelo site quanto pelo app do vendedor e pelo PDV, não precisa reescrever a regra em cada lugar.
**Limitações:**
    1. Prende a lógica de negócio no banco, o que complica testar isso separado da aplicação (é mais difícil simular cenários de teste automatizado).
    2. Esbarra na portabilidade: se um dia a loja resolver trocar de banco de dados, essa procedure toda ia ter que ser reescrita na sintaxe do novo SGBD.
    
**Transações ACID**

Esse é o ponto que realmente resolve o problema dos dois vendedores vendendo o mesmo produto ao mesmo tempo. Se a verificação do estoque e o desconto da quantidade vendida acontecerem dentro da mesma transação (com um nível de isolamento adequado, tipo usando *SELECT ... FOR UPDATE*), o banco garante que a segunda venda só é processada depois que a primeira transação terminar. Assim evita que as duas vendas leiam "1 unidade disponível" ao mesmo tempo e as duas confirmem, deixando o estoque em -1.
    • **Atomicidade:** descontar o estoque e registrar o pedido acontecem juntos, ou nenhum dos dois acontece.
    • **Consistência:** a venda só é confirmada se o estoque continuar válido.
    • **Isolamento:** a venda de um vendedor não vê o estado "no meio do caminho" da venda do outro.
    • **Durabilidade:** depois que a venda é confirmada, ela fica salva mesmo que o servidor caia logo depois.
    
**Vantagens:**
    1. Resolve justamente o nosso problema de concorrência, sem a gente precisar programar manualmente um "lock" de estoque na aplicação.
    2. Simplifica o raciocínio do desenvolvedor sobre falhas, se algo der errado no meio da venda, o banco desfaz tudo sozinho (rollback), sem deixar dado inconsistente pra trás.
**Limitações:**
    1. Um isolamento muito rígido (tipo *SERIALIZABLE*) deixa o sistema mais lento em dias de muita venda, porque os vendedores acabam esperando uma transação terminar pra outra começar.
    2. Se a loja crescer e tiver várias filiais com bancos separados, manter ACID completo entre todos eles fica bem mais difícil (ou caro), é aí que entra a discussão sobre CAP e modelos BASE em bancos NoSQL.

### 1.3 Regras na aplicação

As regras de negócio na aplicação são implementadas no próprio sistema responsável por receber e processar as ações do usuário. Elas normalmente ficam em partes específicas do código, como camadas de serviço, responsáveis por verificar se uma operação pode ou não ser realizada antes de enviar as informações ao banco de dados. Também podem ser utilizadas validações de entrada e recursos oferecidos por frameworks de desenvolvimento. Em um Sistema de Vendas, por exemplo, a aplicação pode verificar se a quantidade solicitada pelo vendedor é menor ou igual à quantidade disponível em estoque. Assim, quando um vendedor tenta realizar uma venda, o sistema consulta o estoque, verifica a regra e somente então permite que a operação continue.

*Vantagens*

Uma das principais vantagens dessa abordagem é a flexibilidade. As regras podem ser alteradas diretamente no código da aplicação, permitindo implementar comportamentos mais complexos e específicos para cada situação. Por exemplo, o sistema pode verificar diferentes condições antes de permitir uma venda, como quantidade disponível, descontos permitidos e perfil do vendedor. Outra vantagem é a facilidade de integração com a interface do sistema. A aplicação pode apresentar imediatamente uma mensagem ao usuário, como “Estoque insuficiente para concluir a venda”, tornando o sistema mais fácil de utilizar.

*Desvantagens*

Por outro lado, existem desvantagens. Uma delas é que a regra pode não ser protegida caso existam várias aplicações acessando o mesmo banco de dados. Se dois vendedores tentarem vender simultaneamente a última unidade de um produto, duas aplicações podem consultar o estoque antes que qualquer uma delas atualize a quantidade. Nesse cenário, apenas a validação na aplicação pode não ser suficiente para garantir a regra de que não seja vendida uma quantidade maior do que a disponível. Outra desvantagem é a possibilidade de duplicação das regras. Se diferentes sistemas ou aplicações acessarem o mesmo banco, cada um pode precisar implementar a mesma validação. Caso uma aplicação seja atualizada e outra não, podem surgir comportamentos diferentes e problemas de consistência.

Portanto, as regras na aplicação são úteis principalmente quando envolvem validações relacionadas ao funcionamento do sistema e à interação com o usuário, oferecendo flexibilidade e facilidade de manutenção. Entretanto, em situações que exigem garantia de consistência dos dados, especialmente em operações simultâneas, somente a aplicação pode não ser suficiente.

### 1.4 Comparativo BD x Aplicação

| Critério                  | Banco de Dados                                                                                                                | Aplicação                                                                                                            |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Consistência              | Permite aplicar as regras de integridade diretamente aos dados, mesmo quando existem diferentes aplicações acessando o banco. | Depende de cada aplicação implementar e seguir corretamente as mesmas regras.                                        |
| Segurança                 | Permite controlar permissões de acesso e restringir operações de acordo com as permissões definidas.                          | Permite controlar acessos, permissões e validar as operações antes de enviá-las ao banco.                            |
| Performance               | Pode ser mais eficiente quando a regra envolve diretamente os dados e pode ser verificada no próprio banco.                   | Pode reduzir consultas desnecessárias quando algumas validações podem ser feitas antes de acessar o banco.           |
| Manutenção                | Facilita a manutenção de regras que precisam ser usadas por várias aplicações, pois ficam centralizadas no banco.             | Pode facilitar a organização e os testes da lógica, mas mudanças podem precisar ser feitas em diferentes aplicações. |
| Portabilidade             | Pode exigir adaptações quando utiliza recursos específicos do SGBD.                                                           | Pode facilitar a mudança de banco quando a lógica não depende diretamente de recursos específicos dele.              |
| Controle central da regra | Permite manter uma regra em um único lugar e aplicá-la às diferentes aplicações que acessam o banco.                          | O controle fica distribuído entre as aplicações, o que pode gerar diferenças na implementação da mesma regra.        |
### 1.5 Análise crítica: qual a melhor opção?

# 2.Exemplos e Casos
Caso real escolhido: Sistema de Vendas
Para ilustrar a diferença entre regra de negócio no banco de dados e na aplicação, o grupo utilizou como caso real um sistema de vendas de uma loja de acessórios de tecnologia (teclado sem fio, fone sem fio, microfone lapela). Em vez de abordar apenas a regra óbvia de controle de estoque, o grupo optou por explorar um cenário mais realista e menos evidente: o que acontece quando dois vendedores tentam vender simultaneamente a última unidade disponível de um mesmo produto. A regra de negócio escolhida foi:
"Não é permitido vender uma quantidade de produto maior do que a disponível em estoque — mesmo quando duas vendas do mesmo produto ocorrem simultaneamente."
Esse cenário é conhecido, na área de banco de dados, como condição de corrida (race condition): situação em que duas operações concorrentes (executadas ao mesmo tempo) podem burlar uma regra de negócio simples caso ela não seja implementada de forma adequada.

# 2.1 Estrutura das tabelas
```sql
CREATE TABLE produtos (
    id_produto SERIAL PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    preco NUMERIC(10,2) NOT NULL CHECK (preco > 0),
    estoque INTEGER NOT NULL CHECK (estoque >= 0)
);

CREATE TABLE vendas (
    id_venda SERIAL PRIMARY KEY,
    id_produto INTEGER NOT NULL REFERENCES produtos(id_produto),
    quantidade INTEGER NOT NULL CHECK (quantidade > 0),
    data_venda TIMESTAMP NOT NULL DEFAULT NOW()
); 
 ```

# 2.2 Exemplo em PostgreSQL — Regra no Banco de Dados
Por que um CHECK simples não é suficiente: um CHECK (estoque >= 0) sozinho impede o estoque de ficar negativo, mas não impede que duas vendas simultâneas leiam o mesmo valor de estoque antes de qualquer uma delas atualizar — permitindo que as duas passem "ao mesmo tempo".
A solução: transformar a checagem e a atualização do estoque em uma única operação atômica (indivisível), usando UPDATE com condição:
```sql
UPDATE produtos
SET estoque = estoque - 1
WHERE id_produto = 3
  AND estoque >= 1;
  ```
  Esse comando só desconta o estoque se, no exato instante da execução, houver unidade disponível. O PostgreSQL garante que nenhuma outra transação consegue "ler" o estoque no meio dessa operação — é tudo ou nada, sem brecha.
Para deixar isso reutilizável como uma função de venda completa, o grupo também implementou uma stored procedure:
```sql
CREATE OR REPLACE FUNCTION registrar_venda(p_id_produto INT, p_quantidade INT)
RETURNS BOOLEAN AS $$
DECLARE
    linhas_afetadas INTEGER;
BEGIN
    UPDATE produtos
    SET estoque = estoque - p_quantidade
    WHERE id_produto = p_id_produto
      AND estoque >= p_quantidade;

    GET DIAGNOSTICS linhas_afetadas = ROW_COUNT;

    IF linhas_afetadas = 0 THEN
        RETURN FALSE; -- estoque insuficiente, venda recusada
    END IF;

    INSERT INTO vendas (id_produto, quantidade) VALUES (p_id_produto, p_quantidade);
    RETURN TRUE; -- venda registrada com sucesso
END;
$$ LANGUAGE plpgsql;
```
## O que essa função garante:
 mesmo que dois vendedores chamem registrar_venda(3, 1) no mesmíssimo milissegundo para o último microfone em estoque, o PostgreSQL processa uma chamada por vez sobre aquela linha. A primeira desconta o estoque e registra a venda; quando a segunda tenta rodar, estoque >= p_quantidade já é falso, ROW_COUNT retorna 0, e a função devolve FALSE — a venda é recusada automaticamente, sem gerar estoque negativo nem venda duplicada.

## 2.3 Exemplo de Validação na Aplicação (Pseudocódigo)
Aqui está o ponto mais importante a destacar no trabalho: a aplicação sozinha não consegue resolver o problema da condição de corrida — ela só consegue fazer uma checagem "otimista" antes de chamar o banco, e depois interpretar corretamente a resposta do banco.

```sql
FUNÇÃO vender_produto(id_produto, quantidade)

    // 1. Checagem inicial na aplicação (rápida, mas não é garantia final)
    produto = buscar_produto(id_produto)

    SE produto não existe ENTÃO
        MOSTRAR "Produto não encontrado."
        PARAR

    SE quantidade <= 0 ENTÃO
        MOSTRAR "Quantidade inválida."
        PARAR

    SE produto.estoque < quantidade ENTÃO
        MOSTRAR "Estoque insuficiente (segundo última leitura)."
        PARAR

    // 2. Chama o banco, que é quem garante a regra de verdade
    sucesso = chamar_banco("registrar_venda", id_produto, quantidade)

    SE sucesso == FALSO ENTÃO
        MOSTRAR "Desculpe, esse item acabou de esgotar. Venda não realizada."
    SENÃO
        MOSTRAR "Venda registrada com sucesso!"

FIM FUNÇÃO
```
Ponto-chave a destacar no trabalho: a checagem feita em SE produto.estoque < quantidade na aplicação é útil para dar um retorno rápido ao vendedor, mas ela não é confiável sozinha — entre o momento em que a aplicação lê o estoque e o momento em que ela manda a venda pro banco, outro vendedor pode ter vendido o mesmo item. Por isso a aplicação precisa sempre checar o retorno do banco (sucesso == FALSO) e tratar essa possibilidade — é o banco, com sua operação atômica, quem garante a regra de verdade.

## 2.4 Aplicando ao caso real da loja
Suponha que a loja tenha apenas 1 unidade de "Microfone lapela" em estoque, e dois vendedores, em caixas diferentes, tentem vender esse produto para dois clientes distintos no mesmo instante:
•	Sem a solução atômica: ambos os sistemas de caixa leem "1 unidade disponível" e confirmam a venda → resultado: 2 vendas registradas para 1 unidade real, estoque fica -1 (inconsistente, prejuízo para a loja).
•	Com a solução implementada (UPDATE ... WHERE estoque >= quantidade dentro da stored procedure): o primeiro caixa a processar a transação vence e desconta o estoque; o segundo recebe FALSE do banco e a aplicação informa ao vendedor que o item acabou de esgotar — nenhuma inconsistência é gerada.
## 2.5 Por que esse exemplo é mais completo que a abordagem simples
Diferente de um exemplo básico que só usa CHECK (estoque >= 0), este caso demonstra que a regra de negócio precisa considerar concorrência — ou seja, múltiplos usuários agindo sobre o mesmo dado ao mesmo tempo, um cenário extremamente comum em sistemas de venda reais (lojas físicas com múltiplos caixas, e-commerces em datas de alta demanda). Isso evidencia que a regra no banco de dados não é apenas "mais uma camada de segurança" — em cenários de concorrência, ela é a única camada capaz de garantir a integridade, já que a aplicação, isoladamente, não tem como saber o que outra instância dela mesma está fazendo no mesmo instante.


## 3. Referências

## 4. Conclusões

## Link do Repositório Git

https://github.com/Livia2896/atividade-regra-negocio-bd
