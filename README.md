# portfolio-cases

Repositório de case studies publicados automaticamente a partir do Zoho Flow.

Cada arquivo em `cases/` é gerado por uma função Deluge que recebe título, cliente e descrição via webhook e escreve o Markdown direto no repositório, usando a Git Data API do GitHub.

## Como funciona

O fluxo tem duas peças:

1. **Gatilho Webhook** no Zoho Flow, que recebe um POST com JSON
2. **Função personalizada `publicarCase`**, que faz seis chamadas HTTP encadeadas ao GitHub

A publicação não usa a Contents API (a rota comum para criar arquivos). Ela monta o commit na mão, objeto por objeto, pelo mesmo caminho que o Git usa internamente:

| Objeto | O que é | Chamada |
|---|---|---|
| **ref** | ponteiro do branch, aponta pro commit mais recente | `GET /git/ref/heads/main` |
| **commit** | aponta pra uma árvore e pro commit anterior | `GET /git/commits/{sha}` |
| **blob** | o conteúdo do arquivo, sem nome | `POST /git/blobs` |
| **tree** | a lista de nomes de arquivo e qual blob cada um aponta | `POST /git/trees` |
| **commit** | o novo commit | `POST /git/commits` |
| **ref** | move o branch pro novo commit | `PATCH /git/refs/heads/main` |

O motivo dessa escolha: a Contents API exige o conteúdo em base64, e `zoho.encryption.base64Encode` não funciona nas funções personalizadas do Zoho Flow. A Git Data API aceita `encoding: "utf-8"` na criação do blob, o que dispensa a codificação.

## Requisitos

**Token do GitHub** — fine-grained personal access token com:

- Repository access: apenas `portfolio-cases`
- Permissions → Contents: **Read and write**
- Permissions → Metadata: Read-only (obrigatório, marcado automaticamente)

O token fica no código, dentro do editor do Zoho. Como ele expira, vale anotar a data de renovação.

## Parâmetros

| Nome | Tipo | Uso |
|---|---|---|
| `titulo` | String | Vira o `# título` do arquivo e o nome do arquivo |
| `cliente` | String | Aparece em negrito logo abaixo do título |
| `descricao` | String | Corpo do case, sob o subtítulo "O que foi feito" |

Payload esperado pelo webhook:

```json
{
  "titulo": "Configuracao de rede em escritorio",
  "cliente": "Cliente particular",
  "descricao": "Levantamento da topologia existente, troca do roteador e segmentacao por VLAN."
}
```

## Saída

O arquivo gerado em `cases/{titulo-em-slug}.md`:

```markdown
# Configuracao de rede em escritorio

**Cliente:** Cliente particular

## O que foi feito

Levantamento da topologia existente, troca do roteador e segmentacao por VLAN.
```

O nome do arquivo vem do título em minúsculas, com espaços trocados por hífen. Publicar com o mesmo título sobrescreve o arquivo em vez de criar outro.

## Particularidades do Deluge no Zoho Flow

Quatro coisas que custaram tempo e valem ficar registradas:

**`zoho.encryption.base64Encode` não funciona.** Retorna erro de execução com mensagem enganosa ("Error occurred while base64Decode response") e número de linha que não corresponde ao código. O resto da família `zoho.encryption` funciona normalmente — `urlDecode`, por exemplo.

**`\n` dentro de string literal não vira quebra de linha.** O Deluge trata como dois caracteres. String em duas linhas o editor recusa. A solução é `zoho.encryption.urlDecode("%0A")`, que devolve o caractere de verdade.

**`invokeurl` usa `parameters` ou `body` para o corpo da requisição, nunca `content`.** Usar uma chave inexistente produz erro de sintaxe apontando linha vizinha. Com string JSON, o header `Content-Type: application/json` é obrigatório — o padrão é `text/plain` e a API recusa com 415.

**`try/catch` não captura falha de API.** O `invokeurl` não lança exceção por status HTTP de erro; ele entrega a resposta normalmente. Um `get("sha")` numa resposta de erro devolve `null` em silêncio e o script segue adiante. A proteção real é verificar o retorno de cada chamada.

## Diagnóstico de erros do GitHub

Um 404 tem duas causas possíveis, e os cabeçalhos da resposta distinguem elas. Para vê-los, adicione `detailed: true` ao `invokeurl`.

- **404 sem `x-ratelimit-*`** — a rota não existe. Erro de digitação na URL.
- **404 com `x-ratelimit-*`** — a rota existe, mas o token não tem permissão. O GitHub responde 404 em vez de 403 para não confirmar a existência do recurso.

Para confirmar que o token está sendo aceito, `GET /repos/{owner}/{repo}` autenticado retorna um bloco `permissions` com `push: true/false`.

Atenção: leitura de repositório **público** não exige autenticação. Chamadas GET funcionam mesmo com token inválido ou ausente — elas não provam nada sobre a credencial.

## Cuidado com `sha: null` na tree

Uma entrada de tree com `sha: null` significa **remover o arquivo**, não é erro. Se o blob falhar e o valor nulo for adiante, o commit apaga o arquivo silenciosamente.

É por isso que a verificação `if (blobSha == null)` precisa vir antes de qualquer uso de `blobSha`. O conteúdo não se perde — commits anteriores mantêm tudo — mas o branch passa a apontar para um estado sem o arquivo.

## Limitações conhecidas

- **Acentos no nome do arquivo** — `Migração` gera `migração.md`. Falta normalizar.
- **Sem guardas na tree e no commit** — só o blob é verificado. Falha nas etapas seguintes ainda passa despercebida e a função retorna `"ok"`.
- **Erros só aparecem no log do Flow** — sem notificação por e-mail.
- **Sem índice** — o README não lista os cases publicados.

## Código

```javascript
string publicarCase(String titulo, String cliente, String descricao)
{
	token = "SEU_TOKEN_AQUI";
	cabecalhos = Map();
	cabecalhos.put("Authorization","Bearer " + token);
	cabecalhos.put("Accept","application/vnd.github+json");
	cabecalhos.put("User-Agent","zoho-deluge-portfolio");
	cabecalhos.put("Content-Type","application/json");
	repo = "https://api.github.com/repos/pesouzaaa/portfolio-cases";
	try
	{
		// 1. Lê o commit mais recente do branch
		refResposta = invokeurl
		[
			url :repo + "/git/ref/heads/main"
			type :GET
			headers:cabecalhos
		];
		commitSha = refResposta.get("object").get("sha");

		// 2. Lê a árvore desse commit
		commitResposta = invokeurl
		[
			url :repo + "/git/commits/" + commitSha
			type :GET
			headers:cabecalhos
		];
		treeSha = commitResposta.get("tree").get("sha");

		// 3. Monta o Markdown
		quebra = zoho.encryption.urlDecode("%0A");
		linhas = List();
		linhas.add("# " + titulo);
		linhas.add("");
		linhas.add("**Cliente:** " + cliente);
		linhas.add("");
		linhas.add("## O que foi feito");
		linhas.add("");
		linhas.add(descricao);
		conteudo = linhas.toString(quebra);

		// 4. Cria o blob (utf-8 dispensa base64)
		corpoBlob = Map();
		corpoBlob.put("content",conteudo);
		corpoBlob.put("encoding","utf-8");
		blobResposta = invokeurl
		[
			url :repo + "/git/blobs"
			type :POST
			parameters:corpoBlob.toString()
			headers:cabecalhos
		];
		blobSha = blobResposta.get("sha");

		if(blobSha == null)
		{
			info "Falha ao criar blob: " + blobResposta;
			return "erro";
		}

		// 5. Cria a árvore, preservando o conteúdo existente via base_tree
		arquivo = Map();
		nomeArquivo = titulo.toLowerCase().replaceAll(" ","-");
		arquivo.put("path","cases/" + nomeArquivo + ".md");
		arquivo.put("mode","100644");
		arquivo.put("type","blob");
		arquivo.put("sha",blobSha);
		listaArquivos = List();
		listaArquivos.add(arquivo);
		corpoTree = Map();
		corpoTree.put("base_tree",treeSha);
		corpoTree.put("tree",listaArquivos);
		treeResposta = invokeurl
		[
			url :repo + "/git/trees"
			type :POST
			parameters:corpoTree.toString()
			headers:cabecalhos
		];
		novaTreeSha = treeResposta.get("sha");

		// 6. Cria o commit encadeado ao anterior
		pais = List();
		pais.add(commitSha);
		corpoCommit = Map();
		corpoCommit.put("message","Adiciona case: " + titulo);
		corpoCommit.put("tree",novaTreeSha);
		corpoCommit.put("parents",pais);
		commitNovo = invokeurl
		[
			url :repo + "/git/commits"
			type :POST
			parameters:corpoCommit.toString()
			headers:cabecalhos
		];
		novoCommitSha = commitNovo.get("sha");

		// 7. Move o ponteiro do branch (force:false impede sobrescrita)
		corpoRef = Map();
		corpoRef.put("sha",novoCommitSha);
		corpoRef.put("force",false);
		refAtualizada = invokeurl
		[
			url :repo + "/git/refs/heads/main"
			type :PATCH
			parameters:corpoRef.toString()
			headers:cabecalhos
		];
		info refAtualizada;
	}
	catch (e)
	{
		info "Erro na linha " + e.lineNo + ": " + e.message;
		return "erro";
	}
	return "ok";
}
```

## Retorno

- `"ok"` — publicado
- `"erro"` — falha capturada; o motivo fica no log de execução do Flow
