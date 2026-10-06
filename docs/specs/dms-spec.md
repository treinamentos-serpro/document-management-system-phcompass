# Especificação - Document Management System

## 1. Objetivo

Permitir que usuários enviem, consultem e baixem seus documentos, mantendo os arquivos no filesystem local da aplicação e os metadados em memória.

## 2. Escopo

### Dentro do escopo

- Enviar um documento e receber seus metadados.
- Listar os documentos pertencentes ao usuário identificado na requisição.
- Baixar um documento pelo identificador, desde que pertença ao usuário identificado.
- Armazenar o conteúdo dos arquivos localmente em `backend/storage`.
- Manter os metadados em memória durante a execução do processo.
- Disponibilizar a interface web para upload, listagem, download e apresentação de erros.

### Fora do escopo

- Armazenamento em nuvem, provedores externos ou serviços de upload de terceiros.
- Persistência de metadados em banco de dados ou recuperação dos metadados após reiniciar o processo.
- Versionamento, edição, compartilhamento ou exclusão de documentos.
- Autenticação, cadastro de usuários e autorização baseada em credenciais reais.
- Busca, filtros avançados, paginação e pré-visualização de conteúdo.

## 3. Premissas e regras de negócio

- O identificador do usuário é fornecido pelo limite HTTP por meio do cabeçalho `X-User-Id`. Nesta fase, trata-se de uma identidade simulada para desenvolvimento, não de autenticação e não de uma garantia de segurança.
- As operações de listagem e download retornam apenas documentos cujo campo `owner` corresponda ao usuário da requisição. A tentativa de acessar documento de outro usuário deve responder como documento não encontrado.
- O nome original é metadado para apresentação e download; nunca deve ser usado como caminho físico do arquivo.
- O nome físico do arquivo deve ser gerado pela aplicação, sem componentes fornecidos pelo cliente. O arquivo fica sob `backend/storage`, utilizando `multer` com `diskStorage`.
- A referência do arquivo físico é interna ao backend e não deve ser incluída nas respostas da API.
- Os metadados são mantidos em memória. Reiniciar o backend remove os registros, mesmo que os arquivos permaneçam no disco; limpeza e recuperação desses arquivos ficam fora do escopo desta fase.
- O upload deve remover o arquivo recém-gravado se a criação do metadado falhar, evitando deixar um arquivo órfão por falha parcial da operação.

## 4. Requisitos funcionais

| ID | Requisito | Critério de aceite |
| --- | --- | --- |
| RF-01 | O usuário pode enviar um único documento usando `multipart/form-data`, no campo `file`. | Com arquivo válido e identidade presente, o sistema grava o conteúdo localmente e responde com os metadados criados. |
| RF-02 | O sistema rejeita upload sem arquivo, identidade ou com arquivo acima do limite configurado. | A resposta indica erro de entrada ou limite excedido e não registra metadados; falha após gravação também remove o arquivo parcial. |
| RF-03 | O usuário pode listar seus documentos. | A resposta contém somente metadados do usuário identificado, sem caminho local ou nome físico; usuário sem documentos recebe lista vazia. |
| RF-04 | O usuário pode baixar documento pelo identificador. | Para documento existente e pertencente ao usuário, o sistema responde com o conteúdo binário e nome original no cabeçalho de download. |
| RF-05 | O sistema não permite que o usuário obtenha documento de outro usuário. | Identificador inexistente e documento pertencente a outro usuário resultam na mesma resposta `404`. |
| RF-06 | A interface apresenta upload, listagem e ação de download, além de estados de carregamento e erro. | O usuário consegue atualizar a lista após upload e recebe mensagens compreensíveis quando uma operação falha. |

## 5. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos enviados são gravados somente no filesystem local em `backend/storage`, usando `multer` com `diskStorage`. |
| RNF-02 | Os metadados ficam em memória nesta fase e não sobrevivem à reinicialização do backend. |
| RNF-03 | Porta, limite de tamanho e demais configurações operacionais devem vir de variáveis de ambiente, com valores padrão documentados quando aplicável. |
| RNF-04 | O backend deve validar entradas HTTP, tratar erros de Multer e de filesystem e não expor detalhes internos, caminhos ou stack traces ao cliente. |
| RNF-05 | O nome original deve ser tratado como dado não confiável ao compor cabeçalhos de resposta; o nome físico deve ser independente dele. |
| RNF-06 | O frontend deve chamar a API com prefixo `/api`; o proxy de desenvolvimento do Vite encaminha essas chamadas para o backend local, removendo o prefixo. |
| RNF-07 | A implementação deve seguir as camadas `routes -> controllers -> services -> repositories`, sem dependência de camadas externas pelas camadas internas. |

### Configuração prevista

| Variável | Finalidade | Padrão proposto |
| --- | --- | --- |
| `PORT` | Porta HTTP do backend. | `3000` |
| `MAX_FILE_SIZE_BYTES` | Tamanho máximo permitido por arquivo. | `10485760` (10 MiB) |
| `STORAGE_DIR` | Diretório local dos arquivos. | `backend/storage`, resolvido relativamente à aplicação e não ao diretório de execução do processo. |

Os valores padrão devem ser definidos e documentados pelo backend. Nenhum provedor externo deve ser configurado ou utilizado.

## 6. Modelo de dados

### Metadados do documento

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `id` | string | Sim | Identificador único e não previsível do documento, gerado pela aplicação. |
| `originalName` | string | Sim | Nome original enviado pelo cliente, preservado como metadado e tratado como conteúdo não confiável. |
| `size` | number | Sim | Tamanho do arquivo em bytes, obtido do arquivo gravado. |
| `uploadedAt` | string | Sim | Data e hora de criação em ISO 8601, em UTC. |
| `owner` | string | Sim | Identificador do usuário associado à requisição. |
| `storageName` | string | Sim, interno | Nome gerado para o arquivo no diretório local; não é exposto pela API. |

`storageName` é informação operacional do repositório, necessária para localizar o conteúdo no disco. As respostas públicas contêm somente `id`, `originalName`, `size`, `uploadedAt` e `owner`.

### Exemplo de metadados públicos

```json
{
  "id": "doc_7f3a9c2e",
  "originalName": "relatorio.pdf",
  "size": 248120,
  "uploadedAt": "2026-10-06T14:30:00.000Z",
  "owner": "usuario-123"
}
```

## 7. Contratos de API

### Convenções

- Rotas do backend: `/upload`, `/documents` e `/documents/:id/download`.
- Prefixo consumido pelo frontend: `/api`; em desenvolvimento, o proxy Vite remove esse prefixo e encaminha ao backend.
- Cabeçalho de identidade para as três operações: `X-User-Id: <identificador>`. Valor ausente ou vazio é inválido. É uma premissa de desenvolvimento e deve ser substituído por identidade estabelecida por autenticação antes de uso real.
- Respostas JSON usam `Content-Type: application/json; charset=utf-8`.
- Formato de erro JSON:

```json
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "O arquivo excede o tamanho máximo permitido."
  }
}
```

### `POST /upload`

- Finalidade: gravar um arquivo localmente e registrar seus metadados.
- Cabeçalhos: `X-User-Id` obrigatório; `Content-Type: multipart/form-data`.
- Campo multipart: `file`, exatamente um arquivo.
- Sucesso: `201 Created`, com metadados públicos do documento.

```json
{
  "id": "doc_7f3a9c2e",
  "originalName": "relatorio.pdf",
  "size": 248120,
  "uploadedAt": "2026-10-06T14:30:00.000Z",
  "owner": "usuario-123"
}
```

- Erros: `400 Bad Request` para identidade ausente ou arquivo ausente/inválido; `413 Payload Too Large` para arquivo acima de `MAX_FILE_SIZE_BYTES`; `500 Internal Server Error` para falha de gravação ou registro. Erros internos não devem revelar caminhos ou stack traces.

### `GET /documents`

- Finalidade: listar os metadados dos documentos do usuário.
- Cabeçalho: `X-User-Id` obrigatório.
- Sucesso: `200 OK`, com array JSON; array vazio quando não há documentos.

```json
[
  {
    "id": "doc_7f3a9c2e",
    "originalName": "relatorio.pdf",
    "size": 248120,
    "uploadedAt": "2026-10-06T14:30:00.000Z",
    "owner": "usuario-123"
  }
]
```

- Erros: `400 Bad Request` quando a identidade está ausente ou vazia; `500 Internal Server Error` para falha inesperada.
- A resposta não inclui `storageName`, caminho físico ou conteúdo do arquivo.

### `GET /documents/:id/download`

- Finalidade: obter o conteúdo binário de um documento do usuário.
- Cabeçalho: `X-User-Id` obrigatório.
- Sucesso: `200 OK`, corpo binário; `Content-Type` correspondente ao tipo detectado ou `application/octet-stream`; cabeçalho `Content-Disposition: attachment` com o nome original codificado de forma segura.
- Erros: `400 Bad Request` para identidade ausente/vazia ou identificador inválido; `404 Not Found` se o documento não existir, pertencer a outro usuário ou seu arquivo não estiver disponível; `500 Internal Server Error` para falha inesperada de leitura.
- A resposta de documento inexistente deve ser indistinguível da tentativa de acesso a documento de outro usuário.

### Códigos de erro previstos

| HTTP | Código | Situação |
| --- | --- | --- |
| `400` | `INVALID_REQUEST` | Identidade, arquivo ou identificador ausente/inválido. |
| `404` | `DOCUMENT_NOT_FOUND` | Documento inexistente, de outro usuário ou sem arquivo legível no local esperado. |
| `413` | `FILE_TOO_LARGE` | Limite configurado de upload excedido. |
| `500` | `INTERNAL_ERROR` | Falha inesperada de filesystem ou processamento. |

## 8. Decisões arquiteturais

### Backend

- `routes/`: registra os métodos e caminhos HTTP e delega para controllers.
- `controllers/`: lê cabeçalhos e parâmetros, realiza validações básicas, traduz chamadas/erros para respostas HTTP e não concentra regras de negócio.
- `services/`: aplica regras de negócio, como associar o dono, garantir escopo por usuário e coordenar gravação e registro de metadados.
- `repositories/`: abstrai o armazenamento local via Multer/diskStorage e a coleção em memória de metadados.
- O fluxo de dependência é `routes -> controllers -> services -> repositories`. As camadas internas não importam Express nem conhecem detalhes HTTP.
- O tratamento de erros de Multer, filesystem e regras de negócio deve produzir os contratos HTTP definidos acima no limite da aplicação.

### Frontend

- Aplicação React com componentes funcionais e Hooks, organizada em `components/`, `pages/` e `services/`.
- A comunicação é feita por `fetch`, concentrada nos serviços do frontend e usando `/api` como prefixo.
- A interface deve apresentar o estado da lista, permitir selecionar/enviar arquivo, atualizar a listagem após sucesso e oferecer download por documento.

### Armazenamento e identidade

- Multer utiliza `diskStorage` apontando para o diretório local configurado; o nome salvo é gerado pela aplicação.
- Metadados permanecem em uma coleção em memória no processo atual; não introduzir banco de dados ou armazenamento remoto.
- `X-User-Id` é exclusivamente uma convenção temporária para desenvolvimento/testes. Implantação com múltiplos usuários reais exige autenticação e obtenção confiável da identidade antes de liberar acesso.

## 9. Plano de execução

Etapas conceituais para implementação futura. Este roteiro não prescreve alterações em arquivos específicos de backend ou frontend.

1. Consolidar contratos, premissas de identidade e limites operacionais; confirmar `MAX_FILE_SIZE_BYTES` e comportamento do nome físico.
2. Implementar o domínio e os repositórios do backend: metadados em memória, gravação local por Multer/diskStorage e tratamento de falhas parciais.
3. Implementar services, controllers e routes conforme as responsabilidades definidas, incluindo isolamento por `owner` e respostas de erro consistentes.
4. Verificar os contratos do backend com testes para upload, listagem, download, isolamento entre usuários, validação, limite de tamanho e falhas de filesystem.
5. Implementar a experiência React para upload, listagem, download, atualização após envio e estados de erro/carregamento, consumindo a API pelo proxy `/api`.
6. Validar a integração ponta a ponta, a configuração por ambiente, o comportamento após reinício e a ausência de dependências de armazenamento externo; registrar limitações conhecidas da persistência em memória.

## 10. Critérios de conclusão

- Todos os requisitos funcionais possuem cobertura por testes ou verificação de integração correspondente.
- Arquivos são gravados apenas localmente por Multer com `diskStorage`; nenhum caminho ou nome físico aparece na API.
- Metadados são mantidos em memória e as respostas/listagens/downloads respeitam o `owner`.
- Os contratos, códigos de erro e o prefixo `/api` estão alinhados entre backend e frontend.
- Variáveis operacionais têm padrão documentado; erros de entrada, Multer e filesystem são tratados sem expor detalhes internos.
- O documento deixa explícita a limitação de identidade simulada e a perda de metadados após reinicialização.