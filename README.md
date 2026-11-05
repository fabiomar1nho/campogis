# CampoGIS

PWA de SIG para trabalho de campo **100% offline**, pensado para celular, em modo escuro e com botões grandes para uso com uma mão ou luvas.

> **Versão beta 0.1.** Funciona, mas ainda não foi validada em campo. Veja [Limitações conhecidas](#limitações-conhecidas) e [Como reportar problemas](#como-reportar-problemas).

Repositório: <https://github.com/fabiomar1nho/campogis>

## O que faz

- **Mapas GeoTIFF** (`.tif` / `.tiff`): ortofotos de drone, cartas digitalizadas, MDE e rasters exportados do QGIS ou ArcGIS.
- **Sobreposições KML / KMZ**: pontos, linhas, polígonos e seus multi-equivalentes, com rótulos legíveis sobre qualquer fundo.
- **GPS offline**: posição, círculo de precisão, seta de rumo e modo seguir posição.
- **Funciona sem internet**: depois da primeira abertura, tudo roda no navegador. Mapas e camadas ficam salvos no aparelho.

## Recursos

### Interface
- Mapa em tela cheia, somente **modo escuro** (economiza bateria em telas OLED).
- Barra de status flutuante: coordenadas, precisão do GPS (±m) e indicador ONLINE/OFFLINE.
- Botões de ação no lado direito: zoom, GPS, rótulos (**Aa**) e camadas (≡).
- Áreas de toque de pelo menos 48 px.
- Toque na coordenada para alternar o formato: **Grau decimal → Graus-min-seg → UTM**.

### GeoTIFF
- Leitura apenas da área visível na tela, em blocos, para poupar memória.
- Resolução original preservada: o app lê os pixels nativos da área visível (até 16 megapixels por vez) e desenha no canvas na densidade real da tela.
- Se o arquivo tiver **overviews (pirâmides)**, o app escolhe o nível adequado ao zoom.
- Controle de opacidade (0 a 100%) por camada.
- Respeita valores NoData.
- Sistemas de coordenadas aceitos:

| Sistema | Códigos EPSG |
|---|---|
| WGS 84 / SIRGAS 2000 geográficas | 4326, 4674, 4258 |
| UTM WGS 84 (norte e sul) | 32601–32660, 32701–32760 |
| UTM SIRGAS 2000 | 31965–31976 (N), 31977–31985 (S) |
| Web Mercator | 3857 |

- Imagens de 1 banda são exibidas em escala de cinza; imagens com 3 ou mais bandas usam as 3 primeiras como RGB.

### KML / KMZ
- Importa `.kml` e `.kmz` (o app lê o `doc.kml` ou o primeiro `.kml` dentro do KMZ).
- Rótulos vindos da tag `<name>`: ao lado do ponto, ou no centroide do polígono.
- Liga e desliga cada camada; botão **Aa** oculta ou mostra todos os rótulos.
- Tocar numa feição abre o **painel inferior** com a tabela de atributos (`ExtendedData`, `Data`, `SimpleData`) e a descrição em HTML sanitizado.
- Botão **📋 COPIAR DESCRIÇÃO** copia o texto para a área de transferência.

### GPS
- Usa `navigator.geolocation.watchPosition` com alta precisão. O GPS do aparelho não precisa de rede.
- Botão ◎: primeiro toque liga o GPS e segue a posição; toques seguintes alternam entre **seguir** e **mapa livre**. Arrastar o mapa solta o modo seguir.
- Seta de rumo usa o `heading` do aparelho ou, se faltar, é calculada pelo deslocamento.

### Persistência
- Arquivos TIFF, KML e KMZ são guardados no **IndexedDB**.
- Ao reabrir, o app restaura camadas, visibilidade, opacidade, posição e zoom do mapa e formato de coordenadas.

## Estrutura

```
campogis/
├── index.html      # aplicação completa (HTML + CSS + JS)
├── sw.js           # Service Worker (cache offline)
├── manifest.json   # manifesto PWA
├── icon.svg        # ícone
└── README.md
```

Bibliotecas, carregadas de CDN e guardadas em cache pelo Service Worker na primeira abertura:

| Biblioteca | Versão | Uso |
|---|---|---|
| Leaflet | 1.9.4 | motor de mapa |
| geotiff.js | 2.1.3 | leitura de GeoTIFF |
| proj4js | 2.9.2 | reprojeção |
| @tmcw/togeojson | 5.8.1 | KML → GeoJSON |
| JSZip | 3.10.1 | leitura de KMZ |

## Instalação e uso

O Service Worker e o GPS exigem **HTTPS** (ou `localhost`).

### Opção 1: GitHub Pages
1. No repositório, vá em **Settings → Pages**.
2. Em *Source*, escolha a branch `main` e a pasta raiz.
3. Acesse `https://fabiomar1nho.github.io/campogis/` pelo celular.

### Opção 2: teste local
```bash
git clone https://github.com/fabiomar1nho/campogis.git
cd campogis
python3 -m http.server 8080
# abra http://localhost:8080
```

### Primeiro uso
1. Abra o app **com internet** e aguarde alguns segundos (o cache das bibliotecas é feito nessa abertura).
2. Recarregue a página uma vez para o Service Worker assumir o controle.
3. Instale na tela inicial (menu do navegador → *Instalar app* / *Adicionar à tela inicial*).
4. Toque em **≡** e importe seus arquivos com **Importar GeoTIFF** ou **Importar KML / KMZ**.
5. Teste o modo avião antes de ir a campo.

### Atualizando
Ao publicar uma nova versão, altere o valor de `V` em `sw.js` (ex.: `campogis-v4`) para forçar a atualização do cache, e recarregue o app duas vezes.

## Preparando seus mapas

- Exporte o GeoTIFF em **WGS 84** ou **UTM** (veja a tabela acima).
- Prefira GeoTIFF **com tiles internos e overviews**. No GDAL:
  ```bash
  gdal_translate -of COG entrada.tif saida.tif
  # ou, para um GeoTIFF comum:
  gdal_translate -co TILED=YES -co COMPRESS=DEFLATE entrada.tif saida.tif
  gdaladdo -r average saida.tif 2 4 8 16 32
  ```
- Para KMZ, o app lê só os dados vetoriais. Ícones e imagens embutidos no KMZ são ignorados.

## Limitações conhecidas

Esta é a primeira beta:

- **Sem mapa de fundo.** Sem GeoTIFF carregado, o mapa fica só com a cor de fundo.
- **CRS não listado** na tabela, ou imagem **rotacionada**, gera mensagem de erro na importação. Reexporte em WGS 84 ou UTM.
- **Arquivos muito grandes sem overviews** podem demorar ao desenhar com o mapa afastado.
- **Acima de 16 MP** na área visível, a leitura é reduzida de forma proporcional para proteger a memória.
- Imagens de 1 banda não têm paleta de cores nem tabela de cores do arquivo (apenas escala de cinza). Imagens com paleta indexada não são suportadas.
- Rótulos só para pontos e polígonos (linhas não têm rótulo).
- Descrição HTML: imagens, scripts, estilos e iframes são removidos por segurança.
- As bibliotecas dependem de CDN **na primeira abertura**. Para eliminar essa dependência, baixe os arquivos para a pasta do projeto e ajuste as URLs em `index.html` e `sw.js`.
- O iOS pode pedir permissão de localização a cada sessão.
- Não testado em todos os navegadores. Priorize Chrome/Edge no Android e Safari no iOS.

## Fora do escopo (por enquanto)

O projeto é propositalmente enxuto. Não estão previstos: edição de geometrias, medições, análise espacial, exportação, sincronização com servidor e mapas de fundo online.

## Como reportar problemas

Abra uma *issue* em <https://github.com/fabiomar1nho/campogis/issues> informando:

- aparelho, sistema e navegador (com versão);
- tipo e tamanho do arquivo (e EPSG, se souber);
- o que aconteceu, o que era esperado e a mensagem de erro exibida;
- se o problema ocorre online, offline ou nos dois.

## Licença

A definir. Enquanto não houver arquivo `LICENSE`, todos os direitos permanecem com o autor.

As bibliotecas de terceiros mantêm suas respectivas licenças.
