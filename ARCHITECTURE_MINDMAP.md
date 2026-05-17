# Architecture Mindmap — Chapter 06: Unit of Work (Exercise)

```
arch-patterns-python
│
├── DOMAIN LAYER  (src/allocation/domain/)
│   └── model.py  — pure business logic, zero infrastructure deps
│       ├── OrderLine  [dataclass, immutable, hashable]
│       │   ├── orderid: str
│       │   ├── sku: str
│       │   └── qty: int
│       │
│       ├── Batch  [aggregate root, mutable]
│       │   ├── reference: str
│       │   ├── sku: str
│       │   ├── eta: Optional[date]
│       │   ├── _purchased_quantity: int
│       │   ├── _allocations: Set[OrderLine]
│       │   ├── allocate(line)
│       │   ├── deallocate(line)
│       │   ├── can_allocate(line) -> bool
│       │   ├── available_quantity  [property]
│       │   └── allocated_quantity  [property]
│       │
│       ├── allocate(line, batches) -> str  [domain service]
│       │   └── sorts batches (in-stock first, then by ETA), picks first available
│       │
│       └── OutOfStock(Exception)
│
├── ADAPTERS LAYER  (src/allocation/adapters/)
│   │
│   ├── orm.py  — SQLAlchemy classical mapping (domain stays ignorant of ORM)
│   │   ├── metadata / engine
│   │   ├── Tables
│   │   │   ├── order_lines  (id, sku, qty, orderid)
│   │   │   ├── batches      (id, reference, sku, _purchased_quantity, eta)
│   │   │   └── allocations  (join table: orderline_id, batch_id)
│   │   └── start_mappers()
│   │       ├── maps OrderLine  → order_lines
│   │       └── maps Batch      → batches (with allocations relationship)
│   │
│   └── repository.py  — Repository pattern (abstract data access)
│       ├── AbstractRepository  [ABC]
│       │   ├── add(batch: Batch)   [abstract]
│       │   └── get(reference) -> Batch  [abstract]
│       │
│       └── SqlAlchemyRepository(AbstractRepository)
│           ├── __init__(session)
│           ├── add(batch)    → session.add(batch)
│           ├── get(ref)      → session.query(Batch).filter_by(reference=ref).one()
│           └── list()        → session.query(Batch).all()
│
├── SERVICE LAYER  (src/allocation/service_layer/)
│   │
│   ├── services.py  — application use cases (orchestration with primitives)
│   │   ├── InvalidSku(Exception)
│   │   ├── is_valid_sku(sku, batches) -> bool
│   │   ├── add_batch(ref, sku, qty, eta, uow)
│   │   │   └── with uow: uow.batches.add(Batch(...)); uow.commit()
│   │   └── allocate(orderid, sku, qty, uow) -> str
│   │       └── with uow: validate SKU → domain.allocate() → uow.commit() → return ref
│   │
│   └── unit_of_work.py  — UoW pattern  ⚠️ EXERCISE — needs implementation
│       │
│       ├── DEFAULT_SESSION_FACTORY  (sessionmaker → PostgreSQL)
│       │
│       ├── AbstractUnitOfWork  [ABC]
│       │   ├── batches: AbstractRepository        ← repository lives here
│       │   ├── commit()   [abstract]
│       │   └── rollback() [abstract]
│       │
│       └── SqlAlchemyUnitOfWork  ← STUB — needs:
│           ├── __init__(session_factory)
│           ├── __enter__()   → open session, assign self.batches = SqlAlchemyRepository(session)
│           ├── __exit__()    → auto-rollback on exception
│           ├── commit()      → session.commit()
│           └── rollback()    → session.rollback()
│
├── ENTRYPOINTS LAYER  (src/allocation/entrypoints/)
│   └── flask_app.py  — HTTP API (outermost layer)
│       ├── POST /add_batch
│       │   └── services.add_batch(ref, sku, qty, eta, SqlAlchemyUnitOfWork())
│       └── POST /allocate
│           ├── services.allocate(orderid, sku, qty, SqlAlchemyUnitOfWork())
│           ├── 201 + {batchref}   on success
│           └── 400 + {message}    on OutOfStock / InvalidSku
│
├── CONFIGURATION  (src/allocation/config.py)
│   ├── get_postgres_uri()  → reads DB_HOST, DB_PASSWORD env vars
│   └── get_api_url()       → reads API_HOST env var
│
└── TESTS  (tests/)
    │
    ├── unit/                          — fast, no I/O
    │   ├── test_batches.py            ✓ Batch domain logic
    │   ├── test_allocate.py           ✓ allocate() domain service
    │   └── test_services.py           ⚠️ EXERCISE
    │       ├── FakeUnitOfWork         ← needs batches attr + committed flag
    │       ├── test_add_batch         ← incomplete assertions
    │       └── 3 × @pytest.mark.skip ← unblock once FakeUnitOfWork is done
    │
    ├── integration/                   — hits real DB (SQLite in-memory or Postgres)
    │   ├── test_orm.py                ✓ ORM mappings round-trip
    │   ├── test_repository.py         ✓ SqlAlchemyRepository CRUD
    │   └── test_uow.py               ⚠️ EXERCISE
    │       ├── test_uow_can_retrieve_and_allocate  ← pytest.fail() design prompt
    │       └── 2 × commented-out rollback tests    ← uncomment after impl
    │
    ├── e2e/                           — requires Docker (Postgres + Flask)
    │   └── test_api.py
    │       ├── test_happy_path_returns_201         ✓
    │       └── test_unhappy_path_returns_400       ✓
    │
    └── conftest.py  — shared fixtures
        ├── in_memory_db    (SQLite, for fast integration tests)
        ├── session_factory / session
        ├── postgres_db     (real Postgres, with retry)
        ├── postgres_session
        └── restart_api     (e2e: restarts Flask process)
```

---

## Layer Dependency Flow

```
 ┌──────────────────────────────────────────────────────────┐
 │  ENTRYPOINTS  (flask_app.py)                             │
 │  — HTTP in/out, creates UoW, calls service functions     │
 └──────────────────────┬───────────────────────────────────┘
                        │ uses
 ┌──────────────────────▼───────────────────────────────────┐
 │  SERVICE LAYER  (services.py + unit_of_work.py)          │
 │  — use cases with primitives, manages transactions       │
 └──────────┬─────────────────────┬────────────────────────-┘
            │ uses                │ uses
 ┌──────────▼──────────┐  ┌──────▼──────────────────────────┐
 │  DOMAIN  (model.py) │  │  ADAPTERS  (repo + orm)         │
 │  — pure business    │  │  — data access, ORM mappings    │
 │    logic, no deps   │  └──────────────┬──────────────────-┘
 └─────────────────────┘                 │ depends on
                                ┌────────▼────────┐
                                │  SQLAlchemy /   │
                                │  PostgreSQL      │
                                └─────────────────┘
```

**Key principle:** Dependency arrows always point *inward* toward the domain.
The domain has zero knowledge of SQLAlchemy, Flask, or any infrastructure.

---

## What the UoW Adds (vs plain Repository)

| Without UoW | With UoW |
|---|---|
| Service receives raw `session` | Service receives abstract `uow` |
| Service calls `session.commit()` | Service calls `uow.commit()` |
| Transaction boundary leaks into service | Transaction fully encapsulated |
| Fake hard to write (need fake session) | `FakeUnitOfWork` trivial (no session needed) |
| Each service manages its own repo | `uow.batches` repo tied to same session/tx |