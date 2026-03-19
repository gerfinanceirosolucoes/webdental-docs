# Decisões de Design (ADRs)

## O que são ADRs?

Architecture Decision Records (ADRs) documentam decisões técnicas importantes e seu contexto. Eles ajudam a entender **por que** algo foi feito de determinada forma.

## Lista de ADRs

| ID | Título | Status | Data |
|----|--------|--------|------|
| [ADR-001](adr-001-cookie-based-sessions.md) | Cookie-based Sessions | ✅ Aceito | 2025-01 |
| [ADR-002](adr-002-valkey-sessoes.md) | Valkey para Sessões | ✅ Aceito | 2025-01 |
| [ADR-003](adr-003-aws-sdk-vs-rest.md) | AWS SDK vs REST | ✅ Aceito | 2025-03 |

## Status possíveis

- ✅ **Aceito** - Decisão em vigor
- 🔄 **Substituído** - Substituído por outra decisão
- ❌ **Rejeitado** - Considerado mas não adotado
- 🟡 **Proposto** - Em discussão

## Como criar um novo ADR

1. Copie o template abaixo
2. Numere sequencialmente (ADR-004, ADR-005, etc)
3. Preencha todas as seções
4. Submeta para revisão

### Template

```markdown
# ADR-XXX: Título da Decisão

## Status
Proposto | Aceito | Substituído | Rejeitado

## Contexto
Qual problema estamos resolvendo? Qual o contexto?

## Decisão
O que decidimos fazer?

## Consequências

### Positivas
- Benefício 1
- Benefício 2

### Negativas
- Trade-off 1
- Trade-off 2

## Alternativas Consideradas

### Alternativa 1
Descrição e motivo de rejeição

### Alternativa 2
Descrição e motivo de rejeição
```
