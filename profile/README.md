# AKS - Agentic Knowledge Systems

Это система, позволяющая быстро искать информацию в динамически меняющемся потоке данных. 

Система предоставляет возможности для быстрого поиска точечной информации на больших объёмах данных. Она глубоко нацелена на работу с ИИ агентами, но также может быть использована напрямую, как показано ниже в примере с приложением. В ней реализовано большое количество функционала: прямое цитирование моделями, обратная связь, отметки об актуальности/свежести источника, оптимизация сложных запросов и множество другого агентоориентирвоанного функционала (например, MCTS, Agentic Pipeline, SLM Router). 

Основная идея - дать ИИ агентам возможность быстро находить нужную информацию, чтобы уменьшить затраты времени, токенов и ресурсов (в ответе возвращается только запрашиваемая информация, а не весь текст документа). За счёт этого можно приблизить качество ответов SLM к LLM, имея максимальную точность и актуальность результатов. Соединение через MCP сервер с мощными агентами (Codex, Claude, Gemini) позволит существенно сократить время ответа и использующийся контекст, что в комбинации с MCTS даст возможность сделать крайне сильную систему оркестрации с самыми точными результатами. 

## Десктопное приложение

<img width="1822" height="1098" alt="Screenshot 2026-07-18 at 19 59 49" src="https://github.com/user-attachments/assets/5363339c-5b0d-4a4d-a7f3-20060ccb3e4e" />

<img width="1822" height="1098" alt="Screenshot 2026-07-18 at 20 00 13" src="https://github.com/user-attachments/assets/d312010f-3eb2-4d71-8fd7-ade265272721" />

## Как верхнеуровнево работает взаимодействие агентов и поискового ядра AKS 

```mermaid
sequenceDiagram
    participant U as User/Agent
    participant R as Router
    participant P as Planner
    participant E as Executor
    participant V as Validator
    participant M as MCTS
    participant S as Synthesizer
    participant A as AKS Core

    U->>R: "Сравни экономики США и Китая"
    R->>R: Классификация → analysis
    R->>P: {type: "analysis", confidence: 0.92}
    P->>P: Декомпозиция на подзапросы
    P->>E: [sub_q1, sub_q2, sub_q3]
    
    par Parallel Search
        E->>A: search(sub_q1)
        E->>A: search(sub_q2)
        E->>A: search(sub_q3)
    end
    
    A-->>E: [results_1, results_2, results_3]
    E->>V: validate(query, merged_results)
    
    alt Low Relevance
        V->>P: needs_rephrase: true
        P->>E: retry with reformulated query
        E->>A: search(reformulated)
        A-->>E: better_results
    end
    
    alt Research Query
        V->>M: activate MCTS
        M->>M: 10 search variations
        M->>A: explore paths
        A-->>M: scored results
        M->>M: UCB selection
        M->>M: expand best path
        M-->>E: combined_results
    end
    
    V-->>S: relevant_results
    S->>S: synthesize answer with citations
    S-->>U: {answer, citations, confidence}
```

## Agentic Pipeline

Система автоматически определяет тип запроса и выбирает оптимальный workflow:

| Тип запроса | Пример | Workflow |
|-------------|--------|----------|
| **fact** | "Столица Франции?" | → прямой поиск |
| **analysis** | "Сравни экономики РФ и КНР" | → декомпозиция → параллельный поиск → синтез |
| **code** | "Где определена authenticate()?" | → code search → граф вызовов |
| **research** | "Комплексный анализ санкций" | → MCTS (10+ вариантов) → итеративное углубление |
| **exploration** | "Расскажи про ИИ" | → multi-search → синтез → self-consistency |

### SLM Router (Qwen-2.5-1.5B + LoRA)
- Классификация запросов на 5 типов
- Датасет: 1000 размеченных примеров
- LoRA fine-tuning через PEFT
- Эвристический fallback если SLM недоступен

### MCTS для research-запросов
```mermaid
graph LR
    Q["research query"] --> V1["variation 1"]
    Q --> V2["variation 2"]
    Q --> V3["variation N"]
    V1 --> S1["search → score"]
    V2 --> S2["search → score"]
    V3 --> S3["search → score"]
    S1 --> UCB["UCB selection"]
    S2 --> UCB
    S3 --> UCB
    UCB --> D1["deeper variation 1"]
    UCB --> D2["deeper variation 2"]
    D1 --> F["final results"]
    D2 --> F
```

## API Endpoints

### Core API
| Method | Path | Описание |
|--------|------|----------|
| `POST` | `/api/v1/search` | Гибридный поиск (dense k-NN + BM25) |
| `POST` | `/api/v1/search/multi` | Мульти-запрос с объединением результатов |
| `POST` | `/api/v1/ingest` | Асинхронная загрузка документа |
| `POST` | `/api/v1/ingest/sync` | Синхронная загрузка |
| `POST` | `/api/v1/feedback` | Обратная связь (helpful/not_helpful) |
| `GET` | `/api/v1/stats` | Статистика базы знаний |
| `POST` | `/api/v1/benchmark` | Offline-бенчмарк |

### Agentic API
| Method | Path | Описание |
|--------|------|----------|
| `POST` | `/api/v1/agentic/search` | Полный agentic pipeline |
| `POST` | `/api/v1/agentic/route` | Классификация запроса |
| `POST` | `/api/v1/agentic/fine-tune` | LoRA fine-tuning SLM |

### System
| Method | Path | Описание |
|--------|------|----------|
| `GET` | `/health/live` | Liveness probe |
| `GET` | `/health/ready` | Readiness probe |
| `GET` | `/metrics` | Prometheus metrics |
| `GET` | `/docs` | Swagger UI |
| `GET` | `/redoc` | ReDoc UI |

### Метрики

<img width="1822" height="1103" alt="Screenshot 2026-07-17 at 11 47 35" src="https://github.com/user-attachments/assets/bd4d7cb0-2199-4a64-9e32-293b891d6ff1" />

