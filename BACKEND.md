# Plano de Backend — Supabase (não implementado)

> **Status:** documentação / roadmap. O sistema atual roda 100% no navegador (`localStorage`).
> Este arquivo descreve **como** plugar um backend quando for necessário. Nada aqui está implementado ainda.

## Quando fazer isto

Não é por volume de acesso (o GitHub Pages aguenta muito tráfego estático). Faça quando precisar de **qualquer** um destes:

- **Dados compartilhados** — várias pessoas trabalhando nos *mesmos* processos e instâncias.
- **Login seguro** — credenciais protegidas, não contornáveis pelo código-fonte.
- **Persistência confiável** — dados que não se perdem ao limpar o navegador ou trocar de dispositivo.
- **Auditoria real** — histórico de quem fez o quê, imutável.

## Arquitetura alvo

O front-end **continua no GitHub Pages**. O Supabase entra **atrás**, como backend:

```
Navegador (index.html no GitHub Pages)
        │
        ├── @supabase/supabase-js  ──►  Supabase Auth      (login)
        │                              Postgres + RLS      (dados)
        │                              Edge Functions      (IA via Groq)
```

- A **anon key** do Supabase **pode** ficar no front-end (é pública por design e protegida por Row Level Security).
- A **service_role key** e a **chave da Groq** **nunca** vão ao front-end — ficam em Edge Functions / variáveis de ambiente.

## 1. Esquema do banco (SQL)

Roda no **SQL Editor** do Supabase. Espelha o modelo de dados atual do app.

```sql
-- PERFIS (espelha auth.users)
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  name text not null,
  email text unique,
  role text default 'Colaborador',
  dept text default 'Administração',
  color text default '#4a9ee8',
  perms text[] default array['view','submit'],
  active boolean default true,
  created_at timestamptz default now()
);

-- PROCESSOS (def + grafo do editor)
create table public.processes (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  dept text,
  sla int default 72,
  color text,
  steps jsonb default '[]',
  nodes jsonb default '[]',
  edges jsonb default '[]',
  created_by uuid references public.profiles(id),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- INSTÂNCIAS (execução)
create table public.instances (
  id uuid primary key default gen_random_uuid(),
  process_id uuid references public.processes(id) on delete cascade,
  title text,
  current_step int default 0,
  status text default 'running',
  assignee uuid references public.profiles(id),
  data jsonb default '{}',
  history jsonb default '[]',
  created_by uuid references public.profiles(id),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- ATIVIDADE / AUDITORIA
create table public.activity (
  id bigint generated always as identity primary key,
  user_id uuid references public.profiles(id),
  verb text, target text, type text,
  created_at timestamptz default now()
);

-- NOTIFICAÇÕES
create table public.notifications (
  id bigint generated always as identity primary key,
  user_id uuid references public.profiles(id),
  title text, body text, icon text,
  read boolean default false,
  created_at timestamptz default now()
);
```

### Criar o perfil automaticamente no cadastro

```sql
create function public.handle_new_user()
returns trigger language plpgsql security definer as $$
begin
  insert into public.profiles (id, name, email)
  values (new.id, coalesce(new.raw_user_meta_data->>'name', new.email), new.email);
  return new;
end; $$;

create trigger on_auth_user_created
after insert on auth.users
for each row execute function public.handle_new_user();
```

## 2. Regras de acesso (RLS) — esboço

Ative o RLS e ajuste as políticas ao seu caso (este é um ponto de partida para uma organização única):

```sql
alter table public.profiles  enable row level security;
alter table public.processes enable row level security;
alter table public.instances enable row level security;

-- helper: o usuário atual tem uma permissão?
-- (use 'admin' = any(perms), 'edit' = any(perms), etc.)

create policy "perfis: ler" on public.profiles
  for select to authenticated using (true);

create policy "perfis: admin gerencia" on public.profiles
  for all to authenticated using (
    exists (select 1 from public.profiles p where p.id = auth.uid() and 'admin' = any(p.perms))
  );

create policy "processos: ler" on public.processes
  for select to authenticated using (true);

create policy "processos: editar" on public.processes
  for all to authenticated using (
    exists (select 1 from public.profiles p where p.id = auth.uid()
            and ('edit' = any(p.perms) or 'admin' = any(p.perms)))
  );

create policy "instancias: ler" on public.instances
  for select to authenticated using (true);

create policy "instancias: operar" on public.instances
  for all to authenticated using (true);
```

## 3. Conectar o front-end

No `index.html`, antes do script principal:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

No início do script:

```js
const SUPABASE_URL  = 'https://SEU_PROJETO.supabase.co';
const SUPABASE_ANON = 'eyJ...';            // chave anon (pública, protegida por RLS)
const sb = supabase.createClient(SUPABASE_URL, SUPABASE_ANON);
```

### O que substituir no código atual

| Hoje (localStorage) | Passa a ser (Supabase) |
|---|---|
| `doLogin()` valida `USERS` | `sb.auth.signInWithPassword({ email, password })` |
| `saveNewUsr()` empurra em `USERS` | admin convida (`sb.auth.admin.inviteUserByEmail` via Edge Function) + insere em `profiles` |
| `loadData()` lê `localStorage` | `sb.from('processes').select()`, `sb.from('instances').select()`… |
| `saveData()` grava `localStorage` | `sb.from(...).upsert(...)` ao salvar/avançar |
| `addActivity()` em array | `sb.from('activity').insert(...)` |

A lógica de **negócio** (motor de roteamento `advanceInstance`, validação, BPMN, simulação) **não muda** — só a camada de leitura/escrita.

## 4. IA (Groq) via Edge Function — tira a chave do navegador

Para um deploy compartilhado, a chave da Groq deixa de ser digitada por usuário e passa para uma função:

```bash
supabase functions new ai-flow
supabase secrets set GROQ_API_KEY=gsk_...
supabase functions deploy ai-flow
```

A função `ai-flow` recebe a descrição, chama a Groq com a `GROQ_API_KEY` (do servidor) e devolve o JSON do fluxo. No front-end:

```js
const { data } = await sb.functions.invoke('ai-flow', { body: { desc } });
applyAIGraph(data);
```

## 5. Checklist de migração (ordem sugerida)

1. Criar projeto no Supabase.
2. Rodar o SQL do esquema (seção 1) e o trigger de perfil.
3. Ativar e ajustar o RLS (seção 2).
4. Criar o usuário admin via **Authentication → Users** e marcar `perms` = `{admin,...}` no `profiles`.
5. Incluir o `supabase-js` e o `createClient` no `index.html` (seção 3).
6. Trocar `doLogin` pela autenticação do Supabase.
7. Trocar `loadData`/`saveData` por consultas/upserts.
8. Trocar criação/edição de usuários pela gestão de `profiles` (admin).
9. Mover a IA para a Edge Function (seção 4).
10. Manter o deploy no **GitHub Pages** — só o backend muda.

## Notas de segurança

- **anon key**: pode ficar no front-end; o que protege os dados é o **RLS**, não esconder a chave.
- **service_role key** e **GROQ_API_KEY**: só no servidor / Edge Functions. Nunca no repositório (veja `.gitignore`).
- Senhas deixam de ficar em texto — o Supabase Auth cuida do hash.
