# SPRINT 5/5 — Triggers, Transações, Integração e Validação Final

**Disciplina:** Laboratório de Banco de Dados  
**Módulo:** 2  
**Modalidade:** Atividade individual  
**Entrega desta Sprint:** `SPRINT5-5.md` + `SPRINT5-5.sql`

---

# Objetivo da Sprint 5/5

Nesta Sprint final do `Module-2`, cada aluno deverá:

1. implementar ao menos um `TRIGGER`;
2. trabalhar com transações;
3. demonstrar `COMMIT`;
4. demonstrar `ROLLBACK`;
5. integrar os conteúdos do módulo;
6. testar todo o código;
7. preparar a entrega final via Pull Request.

Serão trabalhados:

```sql
CREATE TRIGGER
BEFORE INSERT
AFTER INSERT
BEFORE UPDATE
AFTER UPDATE
START TRANSACTION
COMMIT
ROLLBACK
```

---

# 1. Identificação

**Nome completo:**

> NICOLAS LINO OLIVEIRA

**Banco utilizado:**

```text
banco_imobiliaria
```

---

# 2. Revisão do Module-2

| Sprint | Conteúdo | Concluído? |
|---|---|---|
| 1/5 | JOINs | Sim |
| 2/5 | Subconsultas | Sim |
| 3/5 | Views | Sim |
| 4/5 | Procedures e Functions | Sim |
| 5/5 | Triggers e Transações | Sim |

---

# 3. Planejando um TRIGGER

**Regra escolhida:**

> Foram criados dois Triggers:
> 1. **Validação:** uma proposta não pode ter valor menor ou igual a zero (regra de negócio 4 da Sprint 1 do Module 1). O banco bloqueia o `INSERT` inválido, mesmo que ele não venha da Procedure.
> 2. **Automação:** quando um contrato é fechado, o status do imóvel é atualizado sozinho para 'vendido' (venda) ou 'alugado' (aluguel). Isso apoia a regra de negócio 1: um imóvel não pode ficar 'vendido' sem contrato associado.

**Evento:**

- [ ] BEFORE INSERT
- [ ] AFTER INSERT
- [ ] BEFORE UPDATE
- [ ] AFTER UPDATE
- [ ] Outro

> Trigger 1: **BEFORE INSERT**. Trigger 2: **AFTER INSERT**.

**Tabela envolvida:**

```text
Trigger 1: propostas
Trigger 2: contratos (atualiza imoveis)
```

---

# 4. Implementação do TRIGGER

**SQL do seu Trigger:**

```sql
DELIMITER //

CREATE TRIGGER tg_proposta_valida_valor
BEFORE INSERT ON propostas
FOR EACH ROW
BEGIN
    IF NEW.valor_proposto <= 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Valor da proposta deve ser maior que zero.';
    END IF;
END //

CREATE TRIGGER tg_contrato_atualiza_status_imovel
AFTER INSERT ON contratos
FOR EACH ROW
BEGIN
    IF NEW.tipo = 'venda' THEN
        UPDATE imoveis
        SET status = 'vendido'
        WHERE id_imovel = NEW.id_imovel;
    ELSEIF NEW.tipo = 'aluguel' THEN
        UPDATE imoveis
        SET status = 'alugado'
        WHERE id_imovel = NEW.id_imovel;
    END IF;
END //

DELIMITER ;
```

**Explique linha por linha:**

> **Trigger 1 (`tg_proposta_valida_valor`)**
> - `BEFORE INSERT ON propostas`: executa antes de cada nova proposta ser gravada.
> - `FOR EACH ROW`: roda uma vez para cada linha inserida.
> - `NEW.valor_proposto`: `NEW` representa a linha que está sendo inserida.
> - `IF NEW.valor_proposto <= 0`: verifica a regra de negócio.
> - `SIGNAL SQLSTATE '45000'`: gera um erro definido pelo usuário, que cancela o `INSERT`, com a mensagem do `MESSAGE_TEXT`.
>
> **Trigger 2 (`tg_contrato_atualiza_status_imovel`)**
> - `AFTER INSERT ON contratos`: executa depois que o contrato foi gravado com sucesso.
> - `NEW.tipo` e `NEW.id_imovel`: leem o tipo e o imóvel do contrato recém-inserido.
> - `IF` / `ELSEIF`: se for 'venda', o imóvel vira 'vendido'; se for 'aluguel', vira 'alugado'.
> - `UPDATE imoveis ... WHERE id_imovel = NEW.id_imovel`: altera somente o imóvel do contrato.
> - `DELIMITER //`: necessário porque o corpo do Trigger tem vários `;`.

---

# 5. Testando o Trigger

## Trigger 1: validação

**Estado antes do teste:**

```sql
SELECT * FROM propostas WHERE id_cliente = 6;
```

**Operação executada:**

```sql
START TRANSACTION;
CALL sp_registrar_proposta(6, 4, 300000.00);
SELECT * FROM propostas WHERE id_cliente = 6;
ROLLBACK;
```

**Estado depois do teste:**

```sql
SELECT * FROM propostas WHERE id_cliente = 6;
```

**Resultado observado:**

> Esperado: antes, a consulta retorna vazio (a cliente 6 não tem proposta). Com valor válido (R$ 300.000,00), o Trigger deixa passar e a proposta aparece dentro da transação. O `ROLLBACK` desfaz, e a consulta final volta a ficar vazia. Com valor inválido (`-500.00`), o `INSERT` é bloqueado com o erro 1644 e a mensagem "Valor da proposta deve ser maior que zero." (esse teste ficou comentado no `.sql`, pois interrompe o script). *(confirmar no Workbench)*

## Trigger 2: atualização automática do status

**Estado antes do teste:**

```sql
SELECT id_imovel, status FROM imoveis WHERE id_imovel = 2;
```

**Operação executada:**

```sql
INSERT INTO contratos (id_imovel, id_cliente, id_corretor, valor_final, tipo, data_fechamento)
VALUES (2, 3, 2, 750000.00, 'venda', CURDATE());
```

**Estado depois do teste:**

```sql
SELECT
