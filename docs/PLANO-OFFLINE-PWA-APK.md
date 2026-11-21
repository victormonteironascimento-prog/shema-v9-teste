# Shema V9 — Plano de funcionamento offline, sincronização e Android

**Auditoria:** 21/11/2026  
**Branch de trabalho:** `feature/offline-first-pwa-apk`  
**Base auditada:** `main`, arquivo `index.html`, blob `f5ad1c4591f32919487201bd2731d835015005e5`.

## Objetivo e regra de segurança

Entregar a mesma Jornada da Palavra em três formas: navegador, PWA instalável e APK Android. A experiência deve ser local-first: depois da primeira preparação online, leitura e progresso pessoal continuam utilizáveis sem internet; quando a conexão voltar, a sincronização é retomada sem duplicar XP, sobrescrever silenciosamente alterações de outro aparelho ou misturar contas.

**Não publicar em produção, não alterar o banco de produção e não substituir o projeto Supabase configurado até concluir backup, auditoria de schema, migração testada e aprovação explícita.**

## Auditoria do código atual

### O que já existe

- O conteúdo da jornada é montado a partir de `PH` e `EXTRA_TOPICS`; os tópicos adicionais completam as 365 casas. A biblioteca declara 83 casas temáticas e 282 leituras guiadas.
- O progresso é isolado no namespace local `shema-v9-teste::`.
- `PROGRESS_KEYS` já inclui mapas mentais (`shemaMindMaps`), devocionais (`shemaDevotionalEntries`), estudo de livros (`shemaBookStudy`), XP, etapas, avatar, nome, casas desbloqueadas e sequência diária.
- `capture()` e `restore()` usam um envelope de dados `schema: 1` e incluem as notas `shemaNote1...` até `PH.length`.
- A Bíblia Livre (Almeida 1819) já tem cache IndexedDB em `shema-bible-cache-v1`, com armazenamento por capítulo e opção de preparar os 66 livros para leitura offline.
- A autenticação usa Supabase Auth. O código lê e grava um snapshot JSON em `user_progress`; a gravação é agendada após 1,3 s de inatividade.

### Lacunas confirmadas

- O repositório contém apenas o `index.html` na raiz. Não há `manifest.webmanifest`, `service-worker.js`, pacote de dependências, configuração Capacitor ou projeto Android.
- Não há fila persistente de sincronização. Se o `upsert` falha sem internet, o erro é mostrado, mas a alteração pendente não é registrada em uma outbox durável para reenvio posterior.
- Não foram encontrados listeners de reconexão (`online`/`offline`) nem rotina de repetição com backoff.
- O SDK do Supabase é carregado de CDN. Sem cache local desse arquivo, o inicializador de autenticação não consegue iniciar offline.
- O snapshot inteiro é gravado por `upsert`. Em dois aparelhos alterados offline, a última gravação pode substituir a outra; não existe controle de revisão/conflito no cliente auditado.
- O cache da Bíblia é separado do progresso. Essa separação deve continuar: o texto bíblico não deve ser enviado como dado pessoal de sincronização.

### Situação dos ambientes observada

- O projeto Supabase chamado `shema-v9-teste`, apontado pelo HTML auditado, está com status **INACTIVE**. A sincronização real desse ambiente não pode ser validada enquanto ele permanecer inativo.
- Existe outro projeto Supabase, `shema-jornada-da-palavra`, com status **ACTIVE_HEALTHY**. Ele é um ambiente distinto; não deve ser conectado ao V9 por substituição automática.
- A consulta de tabelas do projeto ativo expirou por timeout nesta auditoria. Portanto, a existência, estrutura, constraints e políticas de RLS de `user_progress` ainda precisam ser verificadas antes de qualquer alteração.
- Nenhum ambiente Supabase foi reativado, nenhuma tabela foi alterada e nenhuma publicação de produção foi feita.

## Arquitetura proposta

### 1. PWA e shell offline

Adicionar:
- `manifest.webmanifest` com nome, ícones, cores, orientação e modo standalone;
- `service-worker.js` com cache versionado para shell e arquivos locais;
- SDK Supabase empacotado localmente, em vez de depender exclusivamente de CDN;
- página/fallback offline que informa o estado da conexão e permite abrir o conteúdo previamente armazenado.

Política de cache:
- HTML e assets próprios: estratégia controlada de atualização (network-first para o documento principal, cache-first para assets versionados).
- Bíblia: manter IndexedDB por capítulo; não duplicar o JSON completo no Cache Storage.
- Supabase Auth/REST e respostas privadas: **não** colocar em cache HTTP genérico. A fila de sincronização deve armazenar apenas operações/progresso do usuário autenticado, no IndexedDB privado do dispositivo.
- Atualizações do service worker devem manter a versão anterior funcional até o novo shell instalar e passar no teste de inicialização.

### 2. Persistência local e migração sem perda

- Preservar o namespace atual `shema-v9-teste::` e o formato `capture() schema: 1` durante a primeira fase.
- Criar IndexedDB `shema-offline-v1` com stores separados para `journey`, `outbox`, `metadata` e `device`.
- Na primeira abertura atualizada, copiar o snapshot local para IndexedDB, comparar contagem/chaves e marcar a migração como concluída somente depois da leitura de volta.
- Não apagar localStorage legado automaticamente. Manter exportação JSON e restauração verificável até passar todos os testes.
- Manter a Bíblia em seu banco IndexedDB existente para evitar migração desnecessária de dados volumosos.

### 3. Outbox e sincronização automática

Cada alteração local deve:
1. salvar o snapshot local imediatamente;
2. criar/atualizar uma operação pendente durável na outbox;
3. associar a operação ao `user_id` autenticado, `device_id`, revisão monotônica, horário e chave de idempotência;
4. tentar sincronizar quando houver conexão e sessão válida;
5. marcar como confirmada somente após resposta bem-sucedida do servidor.

A rotina de reconexão deve processar a fila sequencialmente, com backoff exponencial e limite de tentativas, sem descartar dados em erro. Ao trocar de conta, bloquear a fila da conta anterior até autenticação da mesma identidade; nunca enviar o snapshot de um usuário sob outro `user_id`.

### 4. Concorrência entre aparelhos e integridade de XP

Antes de habilitar sincronização multi-dispositivo, escolher e implementar uma estratégia transacional no backend:
- revisão/ETag e compare-and-swap para snapshots, com conflito explícito; e
- eventos idempotentes para conquistas, conclusão de casas e recompensas de XP, para que um retry não conceda XP duas vezes.

Não usar “última gravação vence” como resolução silenciosa. Se duas revisões divergirem, preservar ambas e solicitar/realizar uma mesclagem determinística por campo/evento. As políticas RLS devem restringir leitura e gravação ao próprio usuário.

### 5. APK Android com Capacitor

- Empacotar a mesma aplicação web com Capacitor, sem manter uma segunda implementação da jornada.
- Gerar primeiro APK debug para homologação; depois APK release assinado.
- Guardar o keystore e senhas fora do GitHub e do pacote de build.
- Validar armazenamento local, retorno de rede, deep links de confirmação de e-mail, teclado, rolagem, permissões e atualização.
- Distribuir APK diretamente para testes; preparar AAB/Play Store apenas se essa distribuição for escolhida depois.

## Sequência de implementação

| Etapa | Entrega | Critério de saída |
|---|---|---|
| A — inventário e backup | Inventário de chaves, exportação JSON, teste de restauração | 365 casas, XP, biblioteca, devocionais, mapas e notas conferidos |
| B — PWA | Manifesto, ícones, service worker, SDK local | Instala, abre offline após primeira carga e atualiza sem tela branca |
| C — local-first | IndexedDB de jornada e migração idempotente | Reabrir/atualizar não perde dados nem duplica XP |
| D — outbox | Fila durável, reconexão, backoff, idempotência | Alterar offline, fechar app, reabrir e sincronizar ao voltar a rede |
| E — backend | Schema, RPC/transação, RLS e conflitos | Testes com duas contas e dois aparelhos sem sobrescrita silenciosa |
| F — Capacitor/APK | Projeto Android, APK debug e release | Instalação, atualização e offline validados em Android real |
| G — homologação final | Matriz de regressão e plano de rollback | Aprovação antes de qualquer migração/publicação |

## Matriz obrigatória de regressão

- [ ] As 365 casas continuam disponíveis e na ordem correta.
- [ ] XP e nível permanecem iguais antes/depois da atualização.
- [ ] Conclusão de uma casa/recompensa não duplica XP após retries.
- [ ] Biblioteca (83 casas temáticas + 282 leituras guiadas) permanece completa.
- [ ] Devocionais e mapas mentais permanecem íntegros.
- [ ] Bíblia: 66 livros, capítulos previamente baixados e busca funcionam offline.
- [ ] Notas/reflexões e progresso diário são restaurados.
- [ ] Troca de conta não mistura dados nem outbox.
- [ ] Dois aparelhos offline geram conflito detectável, sem perda silenciosa.
- [ ] Atualização PWA/APK mantém dados locais e oferece rollback.
- [ ] RLS impede acesso cruzado entre usuários.

## Próximo passo técnico imediato

1. Criar backup exportável do snapshot local e validar restauração.
2. Auditar o schema e RLS do projeto Supabase correto; resolver o estado INACTIVE do projeto de teste sem trocar credenciais para produção.
3. Implementar PWA em branch isolada e testar o shell offline antes de mexer na lógica de sincronização.
4. Só depois introduzir outbox e estratégia transacional de conflitos.

**Estado desta branch:** documentação e auditoria inicial apenas. Nenhum código de produção, dado de usuário ou banco foi modificado.
