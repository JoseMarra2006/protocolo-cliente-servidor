# Documentação interativa (Swagger UI)

Esta pasta publica o protocolo como documentação interativa via GitHub Pages,
usando o template [peter-evans/swagger-github-pages](https://github.com/peter-evans/swagger-github-pages)
(MIT — licença original em [`LICENSE-swagger-github-pages`](./LICENSE-swagger-github-pages)).

A página carrega o YAML direto do repositório (`raw.githubusercontent.com`,
branch `main`) — não há cópia do protocolo aqui dentro. O arquivo editado
continua sendo só um: [`protocolo-cliente-servidor.yaml`](../protocolo-cliente-servidor.yaml),
na raiz do repositório.

## Publicar no GitHub Pages

1. No GitHub, ir em **Settings → Pages** do repositório.
2. Em **Build and deployment → Source**, selecionar **Deploy from a branch**.
3. Em **Branch**, selecionar `main` e a pasta **`/docs`**, depois **Save**.
4. Após alguns minutos, a página fica disponível em
   `https://josemarra2006.github.io/protocolo-cliente-servidor/`.

Qualquer push na branch `main` que altere `protocolo-cliente-servidor.yaml`
atualiza a documentação automaticamente (o Swagger UI busca o YAML mais
recente a cada carregamento da página).

## Ver localmente antes de publicar

```bash
python3 -m http.server 8000
```
e abrir `http://localhost:8000/docs/` (rodar o comando na raiz do repositório).
