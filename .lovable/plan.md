
# Plano: Proteger Configuracao da Senha de Exclusao

## Problema Identificado

Atualmente, qualquer pessoa com acesso ao sistema pode:
1. Ir em Configuracoes → Conta
2. Desativar o toggle de "Exigir senha para excluir"
3. Pronto - agora pode excluir agendamentos sem senha

Isso anula completamente a protecao implementada.

## Solucao Proposta

### 1. Exigir Senha Atual para Modificar

Quando a protecao JA estiver ativada:
- Para DESATIVAR: exigir a senha atual
- Para ALTERAR a senha: exigir a senha atual primeiro
- Para acessar a aba CONTA: nao precisa (outras funcoes sao legitimas)

### 2. Recuperacao de Senha via Email

Se o dono esquecer a senha de exclusao:
- Mostrar botao "Esqueci minha senha de exclusao"
- Enviar email para o dono da conta com link para redefinir
- Link valido por 1 hora, uso unico
- Ao clicar, permite definir nova senha de exclusao

## Fluxo Visual

```text
Colaborador tenta desativar protecao
              |
              v
   [Toggle OFF] <-- Clica para desligar
              |
              v
   Modal: "Digite a senha de exclusao"
              |
    [Senha correta?]
         /       \
       SIM       NAO
        |          |
   Desativa    "Senha incorreta"
   protecao    
```

## Recuperacao de Senha

```text
Dono esqueceu a senha de exclusao
              |
              v
   Clica "Esqueci minha senha"
              |
              v
   Email enviado para: dono@email.com
              |
              v
   Link temporario (1h) --> /reset-deletion-password?token=xxx
              |
              v
   Pagina para definir nova senha de exclusao
```

## Mudancas Necessarias

### 1. Banco de Dados

Adicionar colunas para token de recuperacao:

| Coluna | Tipo | Descricao |
|--------|------|-----------|
| `deletion_password_reset_token` | TEXT | Token temporario para reset |
| `deletion_password_reset_expires` | TIMESTAMP | Quando o token expira |

### 2. AccountTab.tsx

Modificar o card de Senha de Exclusao:
- Adicionar validacao de senha ao desativar
- Adicionar validacao de senha ao alterar
- Adicionar link "Esqueci minha senha de exclusao"
- Modal para verificar senha atual antes de qualquer mudanca

### 3. useBusinessSettings.ts

Adicionar novas funcoes:
- `requestDeletionPasswordReset()`: gera token e envia email
- `resetDeletionPasswordWithToken(token, newPassword)`: valida token e define nova senha
- Atualizar `disableDeletionPassword(currentPassword)`: exigir senha

### 4. Edge Function: reset-deletion-password

Nova edge function para:
- Gerar token seguro
- Enviar email com link de recuperacao
- Validar token e permitir reset

### 5. Nova Rota: /reset-deletion-password

Pagina simples para:
- Validar token da URL
- Permitir definir nova senha de exclusao
- Redirecionar para configuracoes apos sucesso

## Interface do Usuario

### Card de Senha de Exclusao (com protecao ativa)

```text
+-----------------------------------------------+
| [Cadeado] Senha de Exclusao                   |
| Proteja a exclusao de agendamentos            |
+-----------------------------------------------+
| [Toggle ON] Exigir senha para excluir         |
|                                               |
| Para alterar esta configuracao, voce          |
| precisara informar a senha atual.             |
|                                               |
| [Alterar Senha]  [Desativar Protecao]         |
|                                               |
| Esqueci minha senha de exclusao               |
+-----------------------------------------------+
```

### Modal de Confirmacao

```text
+-----------------------------------------------+
| Confirme sua senha de exclusao                |
+-----------------------------------------------+
| Para [alterar/desativar] a protecao, digite   |
| sua senha de exclusao atual:                  |
|                                               |
| Senha atual: [........]                       |
|                                               |
| [Cancelar]              [Confirmar]           |
|                                               |
| Esqueci minha senha                           |
+-----------------------------------------------+
```

### Email de Recuperacao

```text
Assunto: Recuperar senha de exclusao - BarberSoft

Ola [Nome da Barbearia],

Voce solicitou a recuperacao da sua senha de exclusao.

Clique no link abaixo para definir uma nova senha:
[BOTAO: Redefinir Senha de Exclusao]

Este link expira em 1 hora.

Se voce nao solicitou esta recuperacao, ignore este email.

Atenciosamente,
Equipe BarberSoft
```

## Detalhes Tecnicos

### Migracao SQL

```sql
ALTER TABLE public.business_settings 
  ADD COLUMN deletion_password_reset_token TEXT,
  ADD COLUMN deletion_password_reset_expires TIMESTAMPTZ;
```

### Edge Function (reset-deletion-password)

```typescript
// POST - Solicitar reset
// Gera token crypto.randomUUID()
// Salva no banco com expiracao de 1 hora
// Envia email via Resend

// PUT - Executar reset
// Valida token e expiracao
// Atualiza deletion_password_hash
// Limpa token
```

### Arquivos a Modificar/Criar

| Arquivo | Acao | Descricao |
|---------|------|-----------|
| Migracao SQL | Criar | Adicionar colunas de reset token |
| `useBusinessSettings.ts` | Modificar | Adicionar funcoes de reset |
| `AccountTab.tsx` | Modificar | Adicionar validacao de senha + link de recuperacao |
| `supabase/functions/reset-deletion-password` | Criar | Edge function para reset |
| `src/pages/ResetDeletionPassword.tsx` | Criar | Pagina para redefinir senha |
| `App.tsx` | Modificar | Adicionar nova rota |

## Seguranca

1. **Token seguro**: UUID v4 gerado por `crypto.randomUUID()`
2. **Expiracao curta**: 1 hora apenas
3. **Uso unico**: Token limpo apos uso
4. **Email verificado**: Enviado apenas para o email do dono da conta
5. **Rate limiting**: Limite de 3 solicitacoes por hora por conta

## Beneficios

- Colaboradores nao podem burlar a protecao
- Dono consegue recuperar caso esqueca a senha
- Processo de recuperacao seguro via email
- Nao bloqueia o acesso as outras configuracoes
