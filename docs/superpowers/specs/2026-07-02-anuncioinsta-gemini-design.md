# Painel de Geração de Anúncios Instagram (anuncioinsta) — Design

## Contexto

A pasta `anuncioinsta/` já contém, hoje, um processo manual: fotos de produtos (bonés
Nike, Mizuno, Hurley, Quiksilver, Lacoste) e vários vídeos de anúncio gerados à mão
(`ad_video_v1` a `v8`, `ad_video_final.mp4`), além de um preview HTML
(`previa_anuncio.html`) que simula como o anúncio aparece no feed/story do Instagram
para a loja **pulse.store26** (Alvorada/RS), com variações de copy (texto principal,
headline, descrição, CTA fixo "Enviar Mensagem").

O objetivo deste projeto é substituir esse processo manual por um painel que gera
imagem, vídeo e copy do anúncio automaticamente via API do modelo
**gemini-omni-flash-preview** (multimodal: imagem + vídeo + texto).

## Objetivo

Painel web local que recebe fotos de produto + informações da campanha e devolve um
pacote de criativos prontos para colar no Meta Ads Manager: imagens (feed 4:5 e 1:1),
vídeo (story 9:16) e variações de copy.

## Arquitetura

Flask síncrono (sem fila de background). Endpoint recebe a requisição, chama a API do
Gemini diretamente, aguarda a resposta e devolve o resultado na mesma request. O
frontend mostra um spinner durante a espera.

Justificativa: volume de geração por sessão é pequeno (2 imagens + 1 vídeo), e os
outros painéis internos do usuário (importação, REINF, certificados) já seguem esse
padrão simples. Se a geração de vídeo se mostrar lenta demais na prática, migrar para
fila em background com polling é a evolução natural — não faz parte deste escopo.

## Estrutura de arquivos

```
anuncioinsta/
  app.py                  # Flask app: rota GET / e rota POST /gerar
  gemini_client.py         # wrapper das chamadas ao gemini-omni-flash-preview
  .env                      # GEMINI_API_KEY (coberto pelo .gitignore raiz: *.env)
  templates/index.html      # painel: formulário + upload + preview do resultado
  campanhas/
    <data>_<slug-marca>/    # uma subpasta por geração
      img_feed_1.png
      img_feed_2.png
      video_story.mp4
      copy.json
```

Os arquivos soltos que já existem em `anuncioinsta/` (fotos e vídeos manuais,
`previa_anuncio.html`) permanecem no lugar, como referência histórica — não são
movidos nem apagados.

Porta do painel: **8759** (as portas 8744, 8753, 8757 e 8758 já estão em uso por
outros painéis do usuário).

## Componentes

- **`templates/index.html`**: formulário com upload múltiplo de fotos, campo de
  marca(s), campo de preço/promoção, campo loja/cidade (pré-preenchido
  `pulse.store26 · Alvorada/RS`, editável) e CTA fixo ("Enviar Mensagem"). Após gerar,
  mostra o resultado reaproveitando o layout de "phone mockup" já usado em
  `previa_anuncio.html` (feed 4:5, story 9:16, feed 1:1) com as imagens/vídeo gerados
  e os cards de copy, cada um com botão de download.
- **`gemini_client.py`**: monta o prompt multimodal (fotos + instruções de marca,
  promoção, loja e tom de voz baseado no padrão já usado no `previa_anuncio.html`) e
  chama o `gemini-omni-flash-preview` pedindo: 2 imagens (feed 4:5 e 1:1), 1 vídeo
  (story 9:16) e copy (3 textos principais, 3 headlines, 2 descrições).
- **`app.py`**: rota `GET /` renderiza o formulário; rota `POST /gerar` recebe fotos e
  campos do formulário, chama `gemini_client`, salva os arquivos em
  `campanhas/<data>_<slug>/` e devolve JSON com os paths gerados e o copy.

## Fluxo de dados

1. Usuário abre `localhost:8759`, preenche o formulário e envia as fotos do produto.
2. Clica em "Gerar". O frontend faz upload via `POST /gerar`.
3. Backend valida a request, monta o prompt e chama a API do Gemini.
4. Enquanto aguarda a resposta, a tela mostra um spinner.
5. Resultado (imagens, vídeo, copy) é salvo em `campanhas/<data>_<slug>/` e devolvido
   como JSON.
6. O painel renderiza o preview no estilo phone mockup com botões de download por
   arquivo.

## Tratamento de erros

- Nenhuma foto enviada → bloqueia o envio no frontend com aviso, sem chamar o backend.
- Falha da API Gemini (rate limit, erro de conteúdo, timeout) → resposta de erro clara
  exibida no painel; a página não trava.
- Falha parcial (ex.: vídeo falha mas as imagens geram com sucesso) → o que deu certo
  é salvo e exibido normalmente; o que falhou aparece com aviso específico, sem
  descartar o resultado parcial.
- Erros são logados no console do Flask para debug local.

## Testes

Ferramenta de uso pessoal/interno. Verificação manual: rodar localmente, subir fotos
reais de produto, conferir que os arquivos saem corretos em `campanhas/` e que o
preview no painel bate com o que foi gerado. Sem suíte automatizada, seguindo o padrão
dos outros painéis internos do usuário.

## Fora de escopo (por agora)

- Fila em background / polling de status.
- Listagem/navegação de campanhas anteriores dentro do painel.
- Publicação automática no Meta Ads Manager (o output continua sendo copiado
  manualmente, como hoje).
