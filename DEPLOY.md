# 🚀 Como publicar no GitHub Pages

## Passo a Passo:

### 1. Criar o repositório no GitHub

1. Acesse: https://github.com/new
2. Nome do repositório: **chatbot**
3. Deixe como **público**
4. **NÃO** adicione README, .gitignore ou licença
5. Clique em **Create repository**

### 2. Configurar o repositório local

No terminal, dentro da pasta do projeto, execute:

```bash
# Inicializar git (se ainda não iniciou)
git init

# Adicionar todos os arquivos
git add .

# Fazer o primeiro commit
git commit -m "Initial commit - Editor de Chatbot"

# Conectar com o repositório remoto
git branch -M main
git remote add origin git@github.com:nathanrab1/chatbot.git

# Enviar o código para o GitHub
git push -u origin main
```

### 3. Fazer o deploy

Execute o script de deploy:

```bash
./deploy.sh
```

Ou, se preferir fazer manualmente:

```bash
# Build do projeto
npm run build

# Entrar na pasta dist
cd dist

# Inicializar git
git init
git add -A
git commit -m "Deploy"

# Enviar para gh-pages
git push -f git@github.com:nathanrab1/chatbot.git main:gh-pages

cd ..
```

### 4. Configurar GitHub Pages

1. Vá em: https://github.com/nathanrab1/chatbot/settings/pages
2. Em **Source**, selecione:
   - Branch: **gh-pages**
   - Folder: **/ (root)**
3. Clique em **Save**

### 5. Acessar o site

Após alguns minutos, seu site estará disponível em:

**https://nathanrab1.github.io/chatbot/**

---

## 🔄 Atualizações futuras

Sempre que fizer mudanças no projeto:

1. Faça commit das suas alterações:
   ```bash
   git add .
   git commit -m "Descrição das mudanças"
   git push
   ```

2. Execute o deploy novamente:
   ```bash
   ./deploy.sh
   ```

---

## ⚠️ Solução de problemas

**Erro de permissão SSH?**
- Configure sua chave SSH: https://docs.github.com/en/authentication/connecting-to-github-with-ssh

**Ou use HTTPS no lugar:**
- Edite o arquivo `deploy.sh` e troque:
  - `git@github.com:nathanrab1/chatbot.git`
  - Por: `https://github.com/nathanrab1/chatbot.git`

**Site não aparece?**
- Aguarde 2-5 minutos após o primeiro deploy
- Verifique se o branch gh-pages foi criado
- Verifique as configurações em Settings > Pages
