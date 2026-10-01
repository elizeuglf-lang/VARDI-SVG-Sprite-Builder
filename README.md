# VARDI SVG Asset Pipeline & Sprite Builder

Ferramenta de arquivo único em **PHP + HTML/CSS/JS nativos** que recebe SVGs, sanitiza, normaliza, otimiza e entrega um `sprites.svg` pronto para usar com `<use>`. Sem Node, sem npm, sem framework, sem bundler.

```
SVG bruto → validação → parse DOM → sanitização → inlining de defs externos
          → namespacing de IDs → normalização de cores → <symbol> → otimização
          → galeria (seleção, edição, duplicatas) → sprite final
```

## Recursos

- **Importação em lote** de sprites (cada `<symbol id="...">` vira um ícone) e de **SVGs avulsos** (viram um ícone com o nome do arquivo), além de **adição manual** colando o código com um ID.
- **Sanitização** de SVG de terceiros (veja [Segurança](#segurança)).
- **Namespacing de IDs**: `<symbolId>--<idOriginal>`. Quatro ícones com `<linearGradient id="grad">` convivem no mesmo sprite sem colisão; `href`, `xlink:href`, `url(#...)`, `begin`/`end` (SMIL), `aria-labelledby` e `aria-describedby` são reescritos junto.
- **Inlining de referências externas**: gradientes, clipPaths etc. definidos fora do `<symbol>` são clonados para um `<defs>` dentro dele, tornando o símbolo autocontido (limite de 5 passadas e 60 clones contra explosão de referências).
- **Normalização de cores**: ícone de uma cor vira `currentColor`; ícone de traço mantém `fill="none"`; ícones multicoloridos, com gradiente, filtro, máscara, pattern ou `url()` em fill/stroke são preservados intactos. Com animação SMIL, a preservação é **por atributo**: só ficam intactos se a animação mexe em atributo de cor (`fill`, `stroke`, `stop-color`, `flood-color`, `lighting-color`, `color`). Um spinner que anima só `transform` ou `opacity` continua obedecendo ao `currentColor` do tema.
- **Estático × animado**: cada ícone é classificado (`animate`, `animateTransform`, `animateMotion`, `animateColor`, `set`) e recebe um selo na galeria.
- **Validação com fallback**: toda otimização é conferida e descartada se quebrar algo (veja abaixo).
- **Relatório de ganho** por ícone e do sprite final.
- **Otimização semântica** (opcional, veja abaixo).
- **Galeria** com filtro por ID, seleção, edição do código de cada ícone e tratamento de **duplicatas**.
- **Exportação** do sprite final, com opção de minificar.

## Requisitos

- PHP **7.0+** (testado em desenho para 7.x/8.x) com as extensões `dom`, `libxml`, `json`, `session`, `ctype`.
- Navegador moderno.
- Nenhuma dependência externa. Foi pensado para rodar também em servidores simples (ex.: AWebServer no Android).

## Como rodar

```bash
php -S localhost:8000
```

Abra `http://localhost:8000/sprite-e.php`.

## Como usar

1. **Mesclar arquivos SVG**: arraste ou selecione arquivos `.svg`. Se o arquivo tiver `<symbol id="...">` (um sprite), cada `<symbol>` vira um ícone. Se não tiver, o `<svg>` inteiro vira **um** ícone, com o ID tirado do nome do arquivo (`home.svg` → `home`; caracteres inválidos viram `-`). Se o nome não render um ID válido, vale o `id` do próprio `<svg>`.
2. **Adicionar manualmente**: informe um ID (ex.: `ic-home`) e cole o código de um `<svg>` avulso.
3. **Selecionar** os ícones desejados (clique nos cards, ou *Todos* / *Nenhum*). Use o campo de filtro para achar por ID.
4. Defina o nome do arquivo, marque/desmarque **Minificar** e clique em **Gerar SVG Final**. O download é um `<nome>.svg`:

```html
<svg xmlns="http://www.w3.org/2000/svg" style="display:none">
  <symbol id="ic-home" viewBox="0 0 24 24">...</symbol>
  ...
</svg>
```

Usando o sprite:

```html
<svg width="24" height="24" aria-hidden="true">
  <use href="sprites-final.svg#ic-home"></use>
</svg>
```

> `<use>` com arquivo externo exige que o sprite seja servido na **mesma origem** da página. Alternativa: cole o conteúdo do sprite inline logo após o `<body>`.

### Ações do card

| Ação | O que faz |
|---|---|
| **Código** / **Salvar edição** | Abre o `<symbol>` para edição; ao salvar, o código passa de novo por todo o pipeline (idempotente: IDs já prefixados não são prefixados outra vez). |
| **Apagar** | Remove o ícone da sessão. |
| **Manter só este** | Em duplicatas, mantém a versão escolhida e descarta as demais. |

### Duplicatas

Ícones com o mesmo ID vindos de arquivos diferentes ficam lado a lado, renomeados com `--2`, `--3`… e destacados na galeria. Na exportação o sufixo é removido e o ícone sai com o ID original; se duas versões do mesmo ID forem selecionadas juntas, a segunda mantém o sufixo para não colidir.

## Otimização semântica

Controlada pelo checkbox **Otimizar** e pelo seletor de casas decimais (1 a 4, ou *sem arredondar*; padrão 3). Vale para o que entrar (importação, adição ou salvamento de edição) **depois** da configuração; ícones já na galeria não são reprocessados. Ao concluir, um aviso mostra os bytes economizados.

| Etapa | Detalhe |
|---|---|
| Metadados de editor | Remove `<metadata>`, elementos/atributos de outros namespaces (`inkscape:*`, `sodipodi:*`) e `data-name`. |
| Atributos redundantes | Remove `opacity="1"` e atributos herdáveis idênticos ao do ancestral mais próximo dentro do símbolo. Não compara com o padrão da especificação, para que os valores explícitos do `<symbol>` continuem protegendo contra CSS global do host. |
| Precisão numérica | Arredonda `d`, `points` e atributos geométricos simples (`x`, `y`, `width`, `height`, `cx`, `cy`, `r`, `rx`, `ry`, `x1`…`y2`) de formas. O parser de `d` entende as *flags* de arco coladas (`a1 1 0 011 1`); se encontrar algo inesperado, deixa o path como está. `transform`, `viewBox`, gradientes e filtros nunca são arredondados. |
| Grupos | `<g>` sem atributos some; `<g>` com um único filho entrega os atributos ao filho. Aborta em conflito (`id`, `clip-path`, `mask`, `filter`, `transform`/`opacity` nos dois). |
| Vazios | Remove `<g>`, `<defs>`, `<title>` e `<desc>` vazios. |
| Órfãos | Remove gradientes, clipPaths, masks, filtros, markers e patterns (e filhos diretos de `<defs>`) que ninguém referencia. |
| IDs sem uso | Remove `id` que nenhum atributo cita (`href`, `url(#)`, SMIL `begin`/`end`, `aria-*`). |

Ícones com **animação SMIL** ou **`<use>`** pulam as etapas estruturais (atributos redundantes e grupos); as demais etapas continuam, sempre preservando qualquer `id` citado por `href`, `url(#)`, `begin`/`end` ou `aria-*`.

### Validação pós-otimização

Depois de otimizar, o `<symbol>` resultante é reparseado e comparado com a versão não otimizada. Se algo piorar, o ícone é guardado **sem otimização** (selo *modo seguro* na galeria). Verificações:

- o XML continua válido;
- nenhuma referência nova ficou quebrada (`url(#)`, `href`/`xlink:href`, `aria-*`, `begin`/`end`);
- nenhum `id` duplicado novo;
- mesma quantidade de elementos de animação e mesmos atributos animados;
- mesma quantidade de formas visíveis e de `<path>` com `d`;
- `viewBox` continua válido.

Problemas que já existiam no original não contam contra a otimização.

### Relatório

Cada card mostra o tamanho antes e depois (`8.42 KB → 5.91 KB (−29.8%)`). Acima da galeria, o resumo do que será exportado: quantidade de ícones, animados e estáticos, tamanho antes/depois e economia. "Antes" é o tamanho do `<symbol>` como veio no arquivo importado (ou do código colado/editado); "depois" é o que vai para o sprite. Com **Otimizar** desligado, "depois" pode ser maior que "antes", porque o namespacing de IDs acrescenta prefixos.

## Segurança

O SVG é tratado como documento ativo, não como imagem inerte. Camadas, na ordem:

1. Limite de tamanho da requisição e **token CSRF** (comparado com `hash_equals`).
2. Bloqueio de `DOCTYPE`/`ENTITY` (XXE) e `LIBXML_NONET`; `libxml_disable_entity_loader` quando disponível.
3. Remoção de elementos perigosos, sem diferenciar maiúsculas: `script`, `iframe`, `object`, `embed`, `link`, `style`, `foreignObject`, `a`, `handler`, `listener`, `image`, `discard`.
4. Remoção de comentários, de todos os atributos `on*` e do atributo `style`.
5. `href`/`xlink:href` só são aceitos como referência interna (`#id`).
6. `url(...)` em qualquer atributo só é aceito como referência interna; o atributo inteiro é removido caso contrário.
7. **SMIL com lista de permissão**: `animate`/`set`/`animateTransform` só podem alterar atributos da lista (nunca `href` ou `on*`), e valores com `javascript:`, `data:` ou `vbscript:` derrubam o elemento.
8. Namespacing de IDs e geração do `<symbol>`.

Sessão: cookie `HttpOnly` e `SameSite=Strict`. O cookie **não** é `Secure` por padrão (a ferramenta costuma rodar em HTTP local); se for servir por HTTPS, adicione `'secure' => true` em `session_set_cookie_params`. Erros PHP são registrados no log e nunca exibidos (`display_errors=0`).

Esta ferramenta **não tem autenticação**. Para expor publicamente, coloque-a atrás de login/HTTPS e de um rate limit no servidor.

## Limites

Definidos por constantes no começo do bloco de tratamento de requisições em `sprite-e.php`:

| Constante | Padrão | Significado |
|---|---|---|
| `VSE_MAX_INPUT_BYTES` | 3 MB | Tamanho máximo de cada requisição. |
| `VSE_MAX_ICONS` | 2000 | Máximo de ícones por sessão. |
| `VSE_MAX_SESSION_BYTES` | 32 MB | Total acumulado de SVG guardado na sessão. |

## Limitações conhecidas

- **Nada é gravado em disco.** Os ícones vivem na sessão PHP; exporte antes de encerrar.
- Arquivo com `<symbol>` é tratado como sprite: um `<svg>` avulso que também contenha símbolos não é importado como ícone à parte.
- O sufixo de duplicata (`--2`, `--3`) é ambíguo: um ícone legitimamente chamado `weather--2` é tratado como duplicata de `weather`.
- `<style>` e `class` são removidos pela sanitização: SVGs que colorem via CSS (ex.: `.cls-1{fill:#f00}`, comum em exportações do Illustrator) perdem essas cores.
- `<image>`, `<a>`, `<foreignObject>` e `<discard>` são removidos.
- Cada resposta devolve todos os ícones da sessão; com milhares de ícones complexos a interface pode ficar pesada.
- Não há suíte de testes automatizados. Teste com seus SVGs antes de usar em produção.

## Estrutura

```
sprite-e.php   aplicação completa (backend PHP + interface)
README.md
LICENSE
```

## Licença

MIT. Veja [LICENSE](LICENSE).
