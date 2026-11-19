# Decisões (estilo ADR)

**Português** · [English](decisions.md)

Registros curtos das decisões que moldaram o portal. Cada uma se baseia no código de produção; o código em si não é publicado.

## 1. Verificar a autorização novamente em cada service, não só na borda

**Contexto.** O Next.js oferece um proxy de requisições (middleware) que pode bloquear rotas antes da renderização. É conveniente, mas ele só enxerga a URL e a sessão, não qual projeto uma consulta vai ler. Um cliente nunca pode ler o projeto de outro cliente.

**Decisão.** O proxy faz uma verificação por rota (usuário autenticado, papel permitido para aquela área). Cada service e route handler então chama `requirePermission` e `requireProjectAccess` contra a tabela `ProjectMember` antes de ler ou gravar. Os repositories não contêm lógica de autorização e nunca são chamados diretamente pelas rotas.

**Consequências.** Algumas verificações rodam duas vezes por requisição. Em troca, adicionar uma nova rota ou um novo consumidor (por exemplo, as ferramentas do assistente do app) não consegue contornar o isolamento entre projetos por acidente: o service chamado já o aplica.

## 2. Um endpoint público, escrito como allowlist com duplo gate

**Contexto.** Uma aplicação separada de mapa público precisa de um subconjunto dos planos de operação de embarcações. Todo o resto do portal exige autenticação.

**Decisão.** Um arquivo de serviço dedicado, sem imports de autenticação, para que quem revisa veja de imediato que a abertura é intencional. A consulta do repository filtra por dois flags independentes (`publicationStatus = PUBLISHED` e `publicVisibility = PUBLISHED_PUBLIC`) e faz `select` de cada campo retornado pelo nome. A resposta é sempre um array, nunca um objeto único.

**Consequências.** Publicar no mapa público é uma ação separada e explícita, distinta de publicar para o cliente. Novas colunas do banco nunca chegam ao endpoint público sem uma mudança de código. O formato da resposta é fixado por testes nas duas aplicações, depois que uma divergência de contrato plano vs. aninhado quebrou o consumidor sem nenhum erro do lado do produtor.

## 3. E-mail por meio de um outbox deduplicado

**Contexto.** As notificações são disparadas a partir de request handlers que podem ser repetidos pelo navegador ou pela plataforma. Enviar diretamente do handler arrisca duplicatas, e ambientes de desenvolvimento nunca devem mandar e-mail para pessoas reais.

**Decisão.** Armazenar cada mensagem primeiro em uma tabela `EmailOutbox` com um `dedupeKey` único e depois tentar a entrega pelo Microsoft Graph. Se nenhum provedor estiver configurado, o registro é marcado como `SKIPPED`.

**Consequências.** No máximo um e-mail por evento lógico, uma trilha de auditoria do que foi enviado e um padrão seguro. A entrega ainda é tentada de forma síncrona, e não por um worker em segundo plano, o que é aceitável no volume atual.

## 4. Recorrer a dados locais apenas em falhas de infraestrutura conhecidas

**Contexto.** As páginas voltadas ao cliente existiam antes do banco de dados e precisavam continuar funcionando enquanto o banco era implantado. Mostrar silenciosamente dados de fixture em produção, porém, seria enganoso.

**Decisão.** Um wrapper `withDatabaseFallback` usado apenas nos caminhos de leitura do cliente. Ele só recorre ao fallback em erros reconhecidos de conexão, autenticação, timeout ou schema ausente, apenas fora de produção, e nunca é usado por mutações administrativas. Qualquer outro erro é propagado.

**Consequências.** As demonstrações continuam funcionando sem banco, e bugs reais não são mascarados. O custo é manter uma pequena lista de assinaturas de erro.

## 5. Deploy manual do build standalone via zip (por enquanto)

**Contexto.** O App Service roda um `node server.js` simples, sem build no servidor configurado, e a aplicação em execução não deve depender da instalação de pacotes no momento do deploy.

**Decisão.** Fazer o build localmente com `output: "standalone"` do Next.js, copiar os assets estáticos para dentro do pacote com um script pós-build e enviar o pacote como um deploy via zip.

**Consequências.** Artefatos previsíveis e nenhuma etapa de build no servidor. O processo depende da máquina e da disciplina de uma pessoa; levá-lo para um pipeline de CI com etapa de aprovação é o principal item em aberto.
