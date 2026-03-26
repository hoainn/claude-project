# Frontend (keepme-react-v4)

## Stack
- React 19.1, TypeScript 5.8, Vite 7.1
- Zustand (state management)
- Zod (validation)
- React Router v7
- TanStack React Table, FullCalendar, Recharts
- CSS Modules + Tailwind

## Dual API Architecture

```
[Frontend]
  ├── Node.js API  →  AUTH only         (NODE_API_BASE_URL)
  └── V4 PHP API   →  all business logic (VITE_API_BASE_URL)
```

Config: `src/config/api.config.ts`

## Directory Structure

```
src/
├── components/     # Reusable UI components (22 dirs)
├── pages/          # Page-level components (22 pages)
├── services/       # API service layer (29+ files, per feature)
├── store/          # Zustand stores (13 stores)
├── hooks/          # Custom React hooks (45+)
├── config/         # api.config.ts (dual API setup)
├── validation/     # Zod schemas
├── utils/          # Utilities
└── constants/      # App-wide constants
```

## Key Patterns

### API Calls
- All requests go through `ApiUtils` static class
- Returns `ServiceResult<T>` for consistent error handling

### Services
Services are exported as singletons and organized by domain:
- `admin/` — dashboard, webhooks, filters, clients
- `sales/` — leads, tasks, CTA, scheduler
- `membership/` — people, tasks, dashboard
- `engagement/` — email, SMS, WhatsApp
- `reports/` — sales, membership, campaigns
- `settings/` — account, users, venues, timezones, team, targets

### Zustand Stores
| Store | Purpose |
|-------|---------|
| `userSessionStore` | Auth, permissions, user data |
| `dropdownOptionsStore` | Cached dropdown data (regions, venues, sources) |
| `notificationStore` | Toast notifications |
| `reportsStore` | Report builder state |
| `themeStore` | Dark/light mode |
| `moduleStore` | Feature module visibility |

## Commands

```bash
npm run dev      # Dev server on port 5173
npm run build    # tsc -b && vite build
npm run lint     # ESLint
```

## Localization
Supports: German, Spanish, French, Portuguese (via constants/localization files)
