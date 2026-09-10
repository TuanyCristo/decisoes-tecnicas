# Decisões técnicas — Concierge

Registro de decisões tomadas durante o desenvolvimento. Atualizar a cada etapa concluída.

## Ambiente

- Projeto local: `C:\Users\tcris\Desktop\concierge` (notebook, Windows).
- Spring Boot 4.1.1, Java 17, Maven.
- Dependências no `pom.xml`: Data JPA, Flyway (+ flyway-database-postgresql), Validation, WebMVC, driver PostgreSQL.
- Spring Security ainda **não** adicionado (entra no passo 2).

## Ordem de construção acordada

1. Estrutura base no banco: tabelas `empresa` e `usuario` via Flyway. **(feito)**
2. Autenticação: Spring Security + JWT, usuário carregado do banco, `empresa_id` no contexto de segurança.
3. Isolamento por empresa: `empresa_id` sempre derivado do usuário autenticado, nunca do request. Coberto por teste automatizado.
4. Módulo Clientes: CRUD completo já sob isolamento.
5. Área de super admin (cadastrar empresas, criar ADMIN da empresa, liberar módulos) via API — por último.

Motivo da ordem: começar pelo super admin exigiria construir autenticação, autorização, CRUD de empresas e módulos contratados antes de qualquer valor de negócio, e adiaria a validação do isolamento (requisito mais crítico).

## Decisões de schema (V1)

- **IDs**: `BIGSERIAL`. Simples e legível. Segurança vem do isolamento, não de ID opaco.
- **Super admin**: `usuario.empresa_id` é nullable; perfil `PLATFORM_ADMIN` não pertence a nenhuma empresa. Alternativa descartada: tabela separada (duplicaria autenticação e login). Vantagem desta escolha: se algum código esquecer o caso, a query filtrando por `empresa_id = NULL` retorna vazio — falha fechada, não vaza dado.
- **Invariante no banco**: `CHECK` garante que `PLATFORM_ADMIN` tem `empresa_id` NULL e qualquer outro perfil tem `empresa_id` preenchido. Não depende do código lembrar.
- **E-mail único global** (`UNIQUE(email)`), não por empresa. Motivo: permite login só com e-mail + senha, sem exigir identificação da empresa. Trade-off: o mesmo e-mail não pode existir em duas empresas. Reversível mais tarde para `UNIQUE(empresa_id, email)` se necessário.
- **Idioma do domínio**: português (`empresa`, `usuario`, `cliente`, `contrato`, `evento`). Vantagem colateral: evita a palavra reservada `user` do PostgreSQL.
- **Exclusão**: sem DELETE físico. Colunas `ativa`/`ativo` para arquivamento.

## Configuração e segredos

- `spring.datasource.username` e `password` vêm de variáveis de ambiente (`DB_USER`, `DB_PASSWORD`), sem valor padrão — a aplicação não sobe sem elas.
- `spring.jpa.hibernate.ddl-auto=validate`. O Flyway é o dono do schema; o Hibernate nunca cria nem altera tabelas.
- `.gitignore` inclui `*-local.properties`, `*-local.sql` e `.env`.
- Usuário super admin será criado no passo 2 por bootstrap lendo variáveis de ambiente (não por migration versionada, para não colocar hash de senha

## Módulos futuros planejados (Prestador, Fornecedor, Ativo, Chamado, Lembrete)

**Status**: apenas desenho registrado. Implementação só após Etapa 2 (autenticação) e Etapa 3 (isolamento).

- **Fornecedor/Prestador**: cadastro próprio, reaproveitando o padrão PF/PJ já usado em `Cliente` (fornecedor é sempre PJ; prestador pode ser PF ou PJ). Existem independente de contrato — cadastrados uma vez, reaproveitados em vários eventos.
- **`ItemContrato`**: representa uma linha de custo dentro de um contrato (ex: "bolo e doces inclusos"). O valor já é conhecido no momento da venda (custo de tabela). O que é restrito não é a existência do valor, e sim sua **visibilidade**: o comercial não deve ver/editar o valor (fora do escopo dele), só o ADM. Requer controle de acesso por perfil (depende da Etapa 2/3). Fornecedor específico é definido depois, na fase de produção — pode ser trocado sem duplicar cadastro, permitindo comparar custo previsto vs. custo real por evento.
- **`Ativo`**: representa um equipamento/instalação do espaço que recebe manutenção ou cuidado (ar condicionado, elevador, extintores, coifa, paisagismo, etc.).
- **`TipoChamado`** e **`TipoLembrete`**: catálogo configurável pela própria empresa (tabela no banco, não Enum fixo) — decisão motivada pelo requisito de a empresa poder cadastrar seus próprios tipos sem depender de alteração de código.
- **`Chamado`** (reativo): problema pontual reportado num `Ativo` (ex: ar condicionado quebrou, vazamento), atendido por um `Prestador`. Não tem recorrência — nasce, é resolvido, encerra.
- **`Lembrete`** (preventivo/recorrente): manutenção ou cuidado agendado num `Ativo`, com periodicidade própria (ex: dedetização, manutenção do elevador, treinamento com nutricionista). Diferente de `Chamado`: é programado, não reativo.

### Pendência em aberto
- Aviso a um `Prestador` sobre disponibilidade de evento (não está ligado a nenhum `Ativo`, é comunicação de oportunidade de negócio) — modelagem adiada, decidir se vira variação de `Chamado` sem `Ativo` obrigatório ou entidade separada, quando a necessidade for concreta.

## Módulo Cliente (entidade)
 
- **Estrutura**: `Cliente` (entidade), `Endereco` e `Contato` (`@Embeddable`), `ValidaDocumento`/`ValidaCpf`/`ValidaCnpj` (validadores stateless, pacote `validadores`).
- **Contato é assimétrico entre PF e PJ**: PF guarda `telefone`/`email` como campos próprios (a pessoa é seu próprio contato). PJ usa `@Embedded Contato` (contato pode ser alguém diferente do responsável legal da empresa). Responsável legal do PJ fica em aberto — será modelado junto de `Contrato`.
- **`@AttributeOverrides` no `Contato` embutido em `Cliente`**: renomeia as colunas para `contato_nome`, `contato_telefone`, `contato_email`, evitando colisão com os campos `telefone`/`email` de PF.
- **Migration V3**: `CHECK` no banco garante que os campos de PF e PJ nunca coexistem no mesmo registro (mesmo padrão já usado para `PLATFORM_ADMIN` em `usuario`) — regra de negócio não depende do código lembrar.
- **Validação de documento**: `pessoaFisica`/`pessoaJuridica` lançam `IllegalArgumentException` se CPF/CNPJ forem inválidos, usando os validadores stateless (sem depender do frontend limpar a entrada).
- **Testado**: `ValidaCpfTest`, `ValidaCnpjTest` (unitários) e `ClienteTest` (integração, confirma exceção lançada com documento inválido) — todos passando contra schema real via `@SpringBootTest`.
- **Pendência consciente**: CRUD completo de Cliente (repository, service, controller, endpoints) está **pausado** até a Etapa 2 (autenticação) e Etapa 3 (isolamento) estarem prontas — expor endpoints agora deixaria `empresaId` sem fonte confiável.