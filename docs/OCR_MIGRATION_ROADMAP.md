# OCR Migration Roadmap — Migração Segura e Incremental

## Objetivos

- Manter o sistema funcional durante toda a migração.
- Definir interfaces e contratos entre Monólito ↔ ACL ↔ OCR.
- Habilitar rollback rápido por feature toggles.
- Garantir qualidade com testes de contrato, integração e E2E.

## Escopo e Referências

- Contrato de API: `docs/OCR_API_CONTRACT.md` (Status: Draft/Proposta).
- Este roadmap trata do controle de tráfego, toggles, rollback e critérios de promoção entre etapas.

## Feature Toggles (variáveis sugeridas)

- `ACL_ENABLED` (bool): Habilita uso da ACL; se `false`, 100% no OCR interno do monólito.
- `ACL_MOCK_MODE` (bool): Respostas simuladas no serviço OCR atrás da ACL (Etapa 1).
- `ACL_TRAFFIC_PERCENT` (0–100): Percentual de requisições roteadas via ACL → OCR novo (Etapa 2+).
- `ACL_FORCE_USERS` (lista): IDs/usuários sempre roteados via ACL (canário controlado).
- `ACL_FORCE_TENANTS` (lista): Tenants sempre roteados via ACL.
- `ACL_USE_WEBHOOK` (bool): Usa webhook para notificar finalização de jobs; caso `false`, usar apenas polling.
- `ACL_TIMEOUT_MS` (número): Timeout de chamadas Monólito ↔ ACL.
- `ACL_RETRY_POLICY` (enum): backoff e limites de retry em falhas (p.ex. `none|simple|exponential`).

## Métricas e Alarmes (SLOs mínimos)

- Taxa de erro 5xx do OCR via ACL (< 1%).
- Latência P95 `POST /ocr/process` (< 2s até aceite; processamento é assíncrono).
- Tempo médio de processamento OCR por página (telemetria/`metrics`).
- Tamanho da fila e tempo em fila (quando houver worker/queue).
- Sucesso de webhooks (taxa de 2xx) e assinaturas inválidas.
- Alarmes: aumento de 5xx, tempo de fila elevado, timeouts, fila parada.

---

## Etapa 1 — ACL com OCR Mockado

- Objetivo: validar contrato, autenticação, observabilidade e fluxo de jobs sem dependência do OCR real.

### Configuração

- `ACL_ENABLED=true`
- `ACL_MOCK_MODE=true`
- `ACL_TRAFFIC_PERCENT=0` (não envia tráfego real do produto; usar apenas testes controlados)
- `ACL_USE_WEBHOOK` opcional (padrão `false` inicialmente; ativar para testar assinatura e segurança)

### Escopo Técnico

- Implementar endpoints da ACL conforme `OCR_API_CONTRACT.md` com respostas mock:
  - `POST /api/v1/ocr/process`: cria `job_id` e retorna `201 accepted`.
  - `GET /api/v1/ocr/status/{job_id}`: evolui estados `processing → done` artificialmente.
  - `POST /api/v1/ocr/webhook` (opcional): valida HMAC/JWT.
  - `GET /api/v1/health`, `GET /api/v1/metrics`: básicos.
- Auth JWT com escopos `documents:write` e `documents:read`.
- Logs estruturados por `job_id`, métricas Prometheus básicas.

### Critérios de Entrada

- OpenAPI/JSON Schemas disponíveis e validados.
- Testes de contrato passando (CDC, se aplicável) contra o mock provider.
- Observabilidade mínima instalada (logs, métricas de requests e erros).

### Critérios de Saída

- Sucesso em testes E2E (felizes e de erro) usando mock.
- Sem regressões no monólito com `ACL_ENABLED=false`.

### Rollback

- Toggle imediato: `ACL_ENABLED=false` (volta 100% ao caminho antigo).
- Não há efeito de dados (mock); limpar memória/estado em caso de inconsistências.

---

## Etapa 2 — OCR funcional em paralelo ao monólito

- Objetivo: processar OCR real via ACL com canário e/ou shadow, mantendo OCR interno disponível como fallback.

### Configuração

- `ACL_ENABLED=true`
- `ACL_MOCK_MODE=false`
- `ACL_TRAFFIC_PERCENT` inicial baixo (p.ex. `5`) e aumentar progressivamente.
- `ACL_FORCE_USERS`/`ACL_FORCE_TENANTS` para grupos piloto.
- `ACL_USE_WEBHOOK=false` inicialmente; habilitar depois de validar segurança (ou manter somente polling).

### Escopo Técnico

- OCR real por worker/queue (ex.: Celery + Redis) chamando Tesseract.
- Persistência de jobs/artefatos e URIs para `text`, `searchable_pdf`, `thumbnail`.
- `status` com progresso e métricas reais (páginas, engine, tempo).
- Observabilidade reforçada: dashboards, tracing, alarmes sobre fila.
- Shadow mode (opcional): executar OCR novo em paralelo sem afetar resposta; comparar qualidade e tempos.

### Incremento de Tráfego

- Aumentar `ACL_TRAFFIC_PERCENT` gradualmente (5 → 20 → 50 → 80).
- Monitorar SLOs a cada incremento; reverter se violados.

### Critérios de Entrada

- Etapa 1 concluída.
- Workers e storage de artefatos operacionais e monitorados.
- Testes de contrato e integração passando com provider real.

### Critérios de Saída

- `ACL_TRAFFIC_PERCENT ≥ 80` por período sustentado sem violações de SLO.
- Qualidade do OCR validada (amostragem e comparações com OCR interno, se aplicável).

### Rollback

- Reduzir `ACL_TRAFFIC_PERCENT` para `0`.
- Manter `ACL_ENABLED=true` apenas se necessário para testes isolados; caso incidente, `ACL_ENABLED=false`.
- Desabilitar `ACL_USE_WEBHOOK` se houver falhas de assinatura/entrega; manter somente polling.

---

## Etapa 3 — Redirecionamento total

- Objetivo: migrar 100% do tráfego para OCR via ACL e desativar OCR interno do monólito.

### Configuração

- `ACL_ENABLED=true`
- `ACL_MOCK_MODE=false`
- `ACL_TRAFFIC_PERCENT=100`
- `ACL_USE_WEBHOOK` habilitado (se maturidade e segurança validadas), ou manter apenas polling.

### Ações

- Confirmar inexistência de dependências ocultas do OCR interno (jobs/sinais/agendamentos).
- Atualizar documentação e comunicação com usuários internos.
- Planejar janela para desativar workers internos com janela de rollback.

### Critérios de Entrada

- Etapa 2 com SLOs sustentados e incidentes resolvidos.
- Cobertura de testes estável (contrato/integrados/E2E/carga).

### Critérios de Saída

- OCR interno desligado sem impacto.
- Observabilidade e alarmes estáveis por período acordado (p.ex. 1–2 semanas).

### Rollback

- Reativar workers internos do monólito.
- Reduzir `ACL_TRAFFIC_PERCENT` (p.ex. para `20` ou `0`).
- Se necessário, `ACL_ENABLED=false` para retorno total ao caminho antigo.

## Roadmap (Gantt)

```mermaid
gantt
    dateFormat  YYYY-MM-DD
    title  Roadmap de Migração OCR (Exemplo)
    section Etapa 1 — ACL Mock
    Definir OpenAPI/Schemas            :done, e1a, 2025-10-20, 7d
    Implementar Endpoints Mock         :e1b, after e1a, 7d
    Observabilidade Básica             :e1c, after e1a, 5d
    Contratos/CI Gates                 :e1d, after e1b, 5d

    section Etapa 2 — OCR Paralelo
    Infra/Fila/Workers                 :e2a, 2025-11-10, 10d
    Artefatos e Métricas Reais         :e2b, after e2a, 7d
    Canary (5%→20%→50%→80%)            :e2c, after e2b, 14d
    Webhook Seguro (opcional)          :e2d, after e2b, 5d

    section Etapa 3 — Cutover Total
    100% Tráfego via ACL               :e3a, 2025-12-10, 3d
    Desativar OCR Interno              :e3b, after e3a, 3d
    Observação Pós-Cutover             :e3c, after e3b, 10d

    section Toggles & Rollback
    ACL_ENABLED / MOCK_MODE            :active, t1, 2025-10-20, 30d
    ACL_TRAFFIC_PERCENT Ramp-up        :t2, 2025-11-20, 20d
    ACL_USE_WEBHOOK                    :t3, 2025-11-25, 15d
