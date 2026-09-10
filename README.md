# Decisões Arquiteturais e Conclusões - Microsserviços de Estoque (PzaaS)

Este documento registra o racional técnico, os trade-offs avaliados e as decisões de design adotadas na construção dos microsserviços de inventário utilizando n8n e PostgreSQL.

---

## 1. Padronização e Resiliência
* **Arquitetura Distribuída e Redundância:** A divisão em microsserviços independentes (`Estoque-A` para baixa e `Estoque-B` / rota geral para insert) com nomenclaturas padronizadas garante o isolamento de falhas, permitindo que problemas em uma operação não comprometam todo o sistema e facilitando os testes de resiliência.

* **Granularidade Padronizada em Gramas:** Optou-se por padronizar todas as unidades de medida internas estritamente como gramas. Essa escolha elimina problemas complexos de arredondamento e conversão de ponto flutuante em operações matemáticas de baixa e reabastecimento.

## 2. Estratégia de Validação (*Fail-Fast*)
* **Validação Visual vs. Lógica de Código:** Decidiu-se utilizar os nós de validação nativos do n8n para a checagem estrutural básica (como verificar se arrays e campos obrigatórios existem). Essa abordagem visual reduz o acoplamento do código e simplifica a manutenção da rotina.

* **Rejeição de Fallbacks Silenciosos:** Estabeleceu-se que o microsserviço não deve mascarar dados incorretos vindos de integrações externas (como preencher valores ausentes com padrões arbitrários). O sistema adota a postura de falhar imediatamente (*fail-fast*), retornando um erro descritivo (ex: `400 Bad Request`) para que a correção seja feita na origem da requisição.

## 3. Persistência e Comportamento do Banco de Dados
* **Processamento Atômico:** Para a baixa de estoque baseada em receitas (B.O.M), a execução de operações transacionais no PostgreSQL garante que a verificação de saldo e a atualização ocorram de maneira atômica, prevenindo condições de concorrência (*race conditions*) e inconsistências de inventário.

* **Atualizações em Lote e Tratamento de Erros:** Para o reabastecimento, utilizou-se uma estratégia de atualização condicional. Além disso, permitiu-se que o banco de dados realize a conversão estrita de tipos, o que viabiliza simular falhas reais de infraestrutura e acionar adequadamente o Erro 500 caso dados corrompidos ou inválidos cheguem à camada de persistência.

* **Auditoria de Ações em Lote:** O uso de variáveis de controle interno do banco de dados permitiu identificar programaticamente se um registro foi inserido ou apenas atualizado durante uma operação em lote, otimizando o feedback transacional sem a necessidade de consultas redundantes.
