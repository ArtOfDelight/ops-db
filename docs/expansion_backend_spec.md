# Expansion tab — backend request

The new **🚀 Expansion** page in `power_dashboard.html` tracks new outlet openings:
per-outlet start / go-live dates, an editable opening checklist (seeded from the
*AOD Cloud Kitchen Opening SOP*), and CAPEX line items. It needs somewhere to
save that. Nothing in `server.py` covers it today, so this asks for 6 small
admin routes, modelled on the `dashboard-comments` routes (`require_admin`,
Firestore, no new env vars or dependencies).

## Collections

| Collection / doc | Holds |
|---|---|
| `expansion_config/template` | The master checklist + default CAPEX lines new outlets are copied from |
| `expansion_projects/{id}` | One doc per outlet being opened |

The SOP content lives in the frontend as the default template, so the backend
has no seed data: `GET template` returns `null` until someone saves one.

### Shapes

```jsonc
// template
{
  "stages": [ { "id": "s1", "name": "Stage 1 – Pre-Opening Preparation",
                "items": [ { "id": "i_ab12", "text": "Apply for FSSAI licence." } ] } ],
  "capex":  [ { "id": "c_ab12", "item": "Security deposit" } ]
}

// project
{
  "id": "…", "name": "HSR Layout", "address": "…",
  "status": "planning" | "in_progress" | "live" | "on_hold",
  "start_date": "2026-10-01", "go_live_date": "2026-11-15",   // YYYY-MM-DD or ""
  "stages": [ { "id": "s1", "name": "…",
                "items": [ { "id": "…", "text": "…", "done": true,
                             "done_at": "2026-10-03", "done_by": "EMP123", "note": "",
                             "assignee_id": "EMP456", "assignee_name": "Rahul" } ] } ],
  "capex":  [ { "id": "…", "item": "Security deposit", "budget": 100000,
                "actual": 90000, "paid": true, "notes": "" } ],
  "created_by": "…", "created_at": "iso", "updated_by": "…", "updated_at": "iso"
}
```

## Routes

| Method | Path | Body → Response |
|---|---|---|
| GET | `/api/admin/expansion/template` | → `{template: {...} \| null}` |
| PUT | `/api/admin/expansion/template` | `{stages, capex}` → `{success}` |
| GET | `/api/admin/expansion/projects` | → `{projects: [...]}` (newest first) |
| POST | `/api/admin/expansion/projects` | full project minus ids/audit → `{success, id}` |
| PUT | `/api/admin/expansion/projects/{id}` | full project (replace) → `{success}`; 404 if missing |
| DELETE | `/api/admin/expansion/projects/{id}` | → `{success}`; 404 if missing |

The frontend always sends the whole project on save (whole-doc replace), so
there's no per-item endpoint. Two admins editing the *same* outlet at the same
moment would be last-write-wins; that seemed fine for this volume.

## Suggested code (drop next to the dashboard-comments routes)

```python
class ExpansionItem(BaseModel):
    id: str
    text: str = Field(..., max_length=500)
    done: bool = False
    done_at: Optional[str] = None
    done_by: Optional[str] = None
    note: Optional[str] = Field(None, max_length=1000)
    assignee_id: Optional[str] = Field(None, max_length=64)      # employee_id the task is assigned to
    assignee_name: Optional[str] = Field(None, max_length=120)   # name at assignment time, for display

class ExpansionStage(BaseModel):
    id: str
    name: str = Field(..., max_length=200)
    items: List[ExpansionItem] = []

class ExpansionCapex(BaseModel):
    id: str
    item: str = Field(..., max_length=200)
    budget: Optional[float] = None
    actual: Optional[float] = None
    paid: bool = False
    notes: Optional[str] = Field(None, max_length=1000)

class ExpansionTemplateRequest(BaseModel):
    stages: List[ExpansionStage] = []
    capex: List[ExpansionCapex] = []

class ExpansionProjectRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=120)
    address: Optional[str] = Field(None, max_length=500)
    status: str = 'planning'              # planning | in_progress | live | on_hold
    start_date: Optional[str] = None      # YYYY-MM-DD or empty
    go_live_date: Optional[str] = None
    stages: List[ExpansionStage] = []
    capex: List[ExpansionCapex] = []


_EXPANSION_STATUSES = {'planning', 'in_progress', 'live', 'on_hold'}

def _expansion_project_data(request: ExpansionProjectRequest) -> Dict:
    """Validate and flatten a project body for Firestore."""
    if request.status not in _EXPANSION_STATUSES:
        raise HTTPException(status_code=400, detail="Invalid status")
    for k in ('start_date', 'go_live_date'):
        v = getattr(request, k)
        if v:
            try:
                datetime.strptime(v, '%Y-%m-%d')
            except ValueError:
                raise HTTPException(status_code=400, detail=f"{k} must be YYYY-MM-DD")
    if len(request.stages) > 50 or sum(len(s.items) for s in request.stages) > 1000 or len(request.capex) > 200:
        raise HTTPException(status_code=400, detail="Too many checklist or CAPEX rows")
    data = request.dict()
    data['start_date'] = data['start_date'] or ''
    data['go_live_date'] = data['go_live_date'] or ''
    return data


def _expansion_ts(d: Dict) -> Dict:
    for k in ('created_at', 'updated_at'):
        ts = d.get(k)
        d[k] = ts.isoformat() if hasattr(ts, 'isoformat') else (str(ts) if ts else None)
    return d


@api_router.get("/admin/expansion/template")
async def get_expansion_template(admin: Dict = Depends(require_admin)):
    """Master opening checklist + default CAPEX lines (null until first saved)."""
    doc = get_firestore_client().collection('expansion_config').document('template').get()
    return {"template": doc.to_dict() if doc.exists else None}


@api_router.put("/admin/expansion/template")
async def put_expansion_template(request: ExpansionTemplateRequest, admin: Dict = Depends(require_admin)):
    get_firestore_client().collection('expansion_config').document('template').set({
        **request.dict(),
        'updated_by': admin['employee_id'],
        'updated_at': firestore.SERVER_TIMESTAMP,
    })
    return {"success": True}


@api_router.get("/admin/expansion/projects")
async def list_expansion_projects(admin: Dict = Depends(require_admin)):
    q = get_firestore_client().collection('expansion_projects') \
        .order_by('created_at', direction=firestore.Query.DESCENDING).limit(200)
    projects = []
    for doc in q.get():
        d = _expansion_ts(doc.to_dict())
        d['id'] = doc.id
        projects.append(d)
    return {"projects": projects}


@api_router.post("/admin/expansion/projects")
async def create_expansion_project(request: ExpansionProjectRequest, admin: Dict = Depends(require_admin)):
    data = _expansion_project_data(request)
    doc_ref = get_firestore_client().collection('expansion_projects').document()
    doc_ref.set({
        **data,
        'created_by': admin['employee_id'], 'created_at': firestore.SERVER_TIMESTAMP,
        'updated_by': admin['employee_id'], 'updated_at': firestore.SERVER_TIMESTAMP,
    })
    return {"success": True, "id": doc_ref.id}


@api_router.put("/admin/expansion/projects/{project_id}")
async def update_expansion_project(project_id: str, request: ExpansionProjectRequest,
                                   admin: Dict = Depends(require_admin)):
    data = _expansion_project_data(request)
    doc_ref = get_firestore_client().collection('expansion_projects').document(project_id)
    if not doc_ref.get().exists:
        raise HTTPException(status_code=404, detail="Project not found")
    doc_ref.update({
        **data,
        'updated_by': admin['employee_id'], 'updated_at': firestore.SERVER_TIMESTAMP,
    })
    return {"success": True}


@api_router.delete("/admin/expansion/projects/{project_id}")
async def delete_expansion_project(project_id: str, admin: Dict = Depends(require_admin)):
    doc_ref = get_firestore_client().collection('expansion_projects').document(project_id)
    if not doc_ref.get().exists:
        raise HTTPException(status_code=404, detail="Project not found")
    doc_ref.delete()
    return {"success": True}
```

Notes for review:
- Uses only `BaseModel`, `Field` and `.dict()`, same as the rest of `server.py`,
  so it works on the current Pydantic without new imports (`datetime` is already used).
- `done_by` is set by the frontend from the logged-in employee; if you'd rather
  the server stamp it, compare old vs new items in the PUT.
- No Firestore index needed (single-field `order_by`).
