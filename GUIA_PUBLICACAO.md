# Guia de Publicação — UniFlow

Passo a passo para colocar o UniFlow no ar. Feito pra quem não programa —
cada passo é clicar em botão e colar código/chave onde indicado.

**Você pode fazer isso agora**, mesmo com o sistema ainda incompleto (faltam
Pendências, Aprovações, Pagamentos, Relatórios). Sempre que eu te entregar
mais uma fase, você repete só o **Passo 5** (2 minutos) para atualizar o site
no ar — não precisa refazer tudo.

---

## O que você vai precisar (contas gratuitas)

| Conta | Para quê | Link |
|---|---|---|
| GitHub | Guardar o código do site | github.com |
| Supabase | Banco de dados + login dos usuários | supabase.com |
| Vercel | Colocar o site no ar | vercel.com |
| OpenAI | A IA que lê as planilhas dos fornecedores | platform.openai.com |

Todas têm plano gratuito suficiente pra 6 usuários internos.

---

## Passo 1 — Criar o banco de dados (Supabase)

1. Entre em **supabase.com** → crie uma conta → **New Project**.
2. Dê um nome (ex: `uniflow-uniembalagens`), crie uma senha de banco (guarde
   essa senha em local seguro) e escolha a região mais próxima (São Paulo/`sa-east-1`).
3. Espere uns 2 minutos o projeto ficar pronto.
4. No menu lateral do Supabase, vá em **SQL Editor** → **New query**.
5. Abra, um por um, os arquivos da pasta `supabase/migrations/` do zip que te
   entreguei (são numerados: `0001_...`, `0002_...` etc.). Copie o conteúdo de
   cada um, cole no SQL Editor, e clique **Run**. **Siga a ordem numérica.**
6. Depois de rodar todas as migrations, faça o mesmo com o arquivo
   `supabase/seed.sql` (isso já cadastra os 12 fornecedores que analisamos).
7. Vá em **Project Settings** (ícone de engrenagem) → **API**. Você vai
   precisar de dois valores agora:
   - **Project URL**
   - **anon public key**

   Guarde os dois — vai usar no Passo 3.

---

## Passo 2 — Subir o código para o GitHub

1. Entre em **github.com** → crie uma conta, se ainda não tiver.
2. Clique em **New repository**. Nome sugerido: `uniflow`. Deixe **Private**
   marcado (é código interno da empresa). Crie.
3. Descompacte o zip do UniFlow que te entreguei no seu computador.
4. Na página do repositório recém-criado, o GitHub mostra um botão
   **"uploading an existing file"** — clique nele, arraste **todo o conteúdo**
   da pasta `uniflow/` (não a pasta em si, o que está dentro dela) e clique
   **Commit changes**.

   *(Se preferir, e tiver o Git instalado, o caminho mais rápido é abrir o
   terminal dentro da pasta `uniflow` e rodar:)*
   ```bash
   git init
   git add .
   git commit -m "Primeira versão"
   git branch -M main
   git remote add origin https://github.com/SEU_USUARIO/uniflow.git
   git push -u origin main
   ```

---

## Passo 3 — Publicar o site (Vercel)

1. Entre em **vercel.com** → **Sign up** → escolha "Continue with GitHub"
   (conecta automaticamente).
2. Clique **Add New → Project**.
3. Selecione o repositório `uniflow` que você acabou de criar → **Import**.
4. Antes de clicar em Deploy, abra **Environment Variables** e adicione:

   | Nome | Valor |
   |---|---|
   | `NEXT_PUBLIC_SUPABASE_URL` | o Project URL do Passo 1 |
   | `NEXT_PUBLIC_SUPABASE_ANON_KEY` | o anon public key do Passo 1 |
   | `NEXT_PUBLIC_APP_URL` | deixe em branco por enquanto, ajustamos depois |
   | `OPENAI_API_KEY` | sua chave da OpenAI (platform.openai.com → API keys → Create new) |
   | `OPENAI_MODEL` | `gpt-4o-mini` |

5. Clique **Deploy**. Espera de 1 a 3 minutos.
6. Quando terminar, a Vercel te dá um endereço tipo
   `https://uniflow-xxxx.vercel.app`. **Copie esse endereço.**
7. Volte em Environment Variables, edite `NEXT_PUBLIC_APP_URL` colando esse
   endereço, e clique em **Redeploy** (na aba Deployments) pra aplicar.

Pronto — o site está no ar.

---

## Passo 4 — Criar o primeiro usuário administrador

1. Acesse o site publicado, vá até a tela de login. Como ainda não existe
   tela de "criar conta" pública (é um sistema interno — cadastro é feito
   pelo admin), crie o primeiro usuário direto pelo Supabase:
   - No painel do Supabase → **Authentication → Users → Add user**.
   - Preencha e-mail e senha, marque **Auto Confirm User**.
2. Volte no **SQL Editor** do Supabase e rode, trocando pelo e-mail que você
   acabou de criar:
   ```sql
   update public.profiles
   set role = 'admin'
   where id = (select id from auth.users where email = 'seuemail@uniembalagens.com.br');
   ```
3. Faça login no site com esse e-mail e senha. Você deve cair no Dashboard,
   com o menu completo (Cadastros, Importação, etc.) visível.

---

## Passo 5 — Atualizar o site quando eu te entregar uma nova fase

Toda vez que eu te mandar um novo zip com mais funcionalidades:

1. Descompacte o novo zip.
2. Repita o upload no GitHub (Passo 2) **substituindo os arquivos** — ou, se
   estiver usando Git pelo terminal, é só rodar de novo `git add .`,
   `git commit`, `git push`.
3. A Vercel detecta o push automaticamente e republica o site sozinha — não
   precisa fazer mais nada.
4. Se eu tiver adicionado uma nova migration de banco (arquivo novo em
   `supabase/migrations/`), rode só esse arquivo novo no SQL Editor do
   Supabase, do mesmo jeito do Passo 1.

---

## Dúvidas comuns

**"Dá erro ao processar planilha, mencionando OPENAI_API_KEY."**
Confirme se a variável `OPENAI_API_KEY` foi mesmo salva na Vercel e se você
clicou em Redeploy depois de salvar (variáveis de ambiente só entram em
vigor no próximo deploy).

**"Esqueci minha senha de banco do Supabase."**
Sem problema, ela só é pedida se você quiser conectar por linha de comando.
Pelo painel web (SQL Editor, Authentication, etc.) você não precisa dela.

**"Posso usar um domínio da empresa (ex: comissoes.uniembalagens.com.br) em
vez do endereço .vercel.app?"**
Sim — na Vercel, vá em **Project → Settings → Domains** e siga as instruções
para apontar o domínio (isso exige acesso ao DNS do domínio da empresa).
