# SignalR Real-time User Story - Full Sample (React + C#)

This repository contains a minimal but complete example showing how to implement **real-time synchronization** for a work item (user story) using **ASP.NET Core (C#)** with **SignalR** on the backend and **React** on the frontend (SignalR client). The sample uses an in-memory store for simplicity.

---

## File tree

```
/README.md
/backend/
  Program.cs
  WorkItem.cs
  WorkItemRepository.cs
  WorkItemHub.cs
  Controllers/
    WorkItemsController.cs
/frontend/
  package.json
  vite.config.js
  src/
    main.jsx
    App.jsx
    WorkItemView.jsx
    api.js
    index.css
```

---

## README (usage)

```md
# SignalR Real-time User Story Example

## Prerequisites
- .NET 8 SDK (or .NET 7 - adjust target framework in csproj)
- Node 18+

## Backend
1. `cd backend`
2. `dotnet run`
   - By default runs on `http://localhost:5000` and `https://localhost:5001` (Kestrel). Adjust launch settings or the `urls` environment variable as needed.

## Frontend
1. `cd frontend`
2. `npm install`
3. `npm run dev` (Vite) or `npm start` if you adapt to CRA
4. Open browser to the address printed by Vite (e.g. `http://localhost:5173`)

## How it works
- Frontend loads a list of work items and can open a single work item view.
- When user updates the State on the frontend, frontend sends a REST `PUT` to backend.
- Backend updates the item and notifies all connected clients via SignalR.
- Clients subscribed to that item's group receive the `WorkItemUpdated` event and update UI instantly.

```

---

## Backend: Program.cs

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyHeader()
              .AllowAnyMethod()
              .WithOrigins("http://localhost:5173") // change if frontend runs elsewhere
              .AllowCredentials();
    });
});

builder.Services.AddControllers();
builder.Services.AddSignalR();

// Simple in-memory repository
builder.Services.AddSingleton<WorkItemRepository>();

var app = builder.Build();

app.UseCors();
app.MapControllers();

app.MapHub<WorkItemHub>("/hubs/workitems");

app.Run();
```

## Backend: WorkItem.cs

```csharp
public class WorkItem
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public string State { get; set; } = "New"; // e.g. New, Active, Resolved, Closed
}
```

## Backend: WorkItemRepository.cs

```csharp
using System.Collections.Concurrent;

public class WorkItemRepository
{
    private readonly ConcurrentDictionary<int, WorkItem> _store = new();
    private int _nextId = 1;

    public WorkItemRepository()
    {
        // seed some items
        Add(new WorkItem { Title = "User Story: Login", Description = "Allow users to log in" });
        Add(new WorkItem { Title = "User Story: Profile", Description = "Profile page" });
    }

    public IEnumerable<WorkItem> GetAll() => _store.Values.OrderBy(w => w.Id);
    public WorkItem? Get(int id) => _store.TryGetValue(id, out var w) ? w : null;

    public WorkItem Add(WorkItem item)
    {
        var id = Interlocked.Increment(ref _nextId);
        item.Id = id;
        _store[id] = item;
        return item;
    }

    public WorkItem? UpdateState(int id, string newState)
    {
        if (_store.TryGetValue(id, out var item))
        {
            item.State = newState;
            return item;
        }
        return null;
    }
}
```

## Backend: WorkItemHub.cs

```csharp
using Microsoft.AspNetCore.SignalR;

public class WorkItemHub : Hub
{
    // Clients will join group per workitem so only relevant clients get updates
    public Task JoinGroup(string workItemId)
        => Groups.AddToGroupAsync(Context.ConnectionId, GroupName(workItemId));

    public Task LeaveGroup(string workItemId)
        => Groups.RemoveFromGroupAsync(Context.ConnectionId, GroupName(workItemId));

    private static string GroupName(string id) => $"workitem-{id}";
}
```

## Backend: Controllers/WorkItemsController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.SignalR;

[ApiController]
[Route("api/[controller]")]
public class WorkItemsController : ControllerBase
{
    private readonly WorkItemRepository _repo;
    private readonly IHubContext<WorkItemHub> _hub;

    public WorkItemsController(WorkItemRepository repo, IHubContext<WorkItemHub> hub)
    {
        _repo = repo;
        _hub = hub;
    }

    [HttpGet]
    public IActionResult GetAll() => Ok(_repo.GetAll());

    [HttpGet("{id:int}")]
    public IActionResult Get(int id)
    {
        var item = _repo.Get(id);
        if (item == null) return NotFound();
        return Ok(item);
    }

    // Update state endpoint
    [HttpPut("{id:int}/state")]
    public async Task<IActionResult> UpdateState(int id, [FromBody] UpdateStateRequest req)
    {
        var updated = _repo.UpdateState(id, req.State);
        if (updated == null) return NotFound();

        // Notify clients in the group's workitem
        await _hub.Clients.Group($"workitem-{id}").SendAsync("WorkItemUpdated", updated);

        return Ok(updated);
    }
}

public record UpdateStateRequest(string State);
```

---

## Frontend: package.json

```json
{
  "name": "workitem-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "@microsoft/signalr": "7.0.5"
  },
  "devDependencies": {
    "vite": "5.0.0",
    "@vitejs/plugin-react": "4.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

## Frontend: vite.config.js

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

## Frontend: src/main.jsx

```jsx
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './index.css'

createRoot(document.getElementById('root')).render(<App />)
```

## Frontend: src/App.jsx

```jsx
import React, { useEffect, useState } from 'react'
import { getAllWorkItems } from './api'
import WorkItemView from './WorkItemView'

export default function App(){
  const [items, setItems] = useState([])
  const [selected, setSelected] = useState(null)

  useEffect(()=>{
    getAllWorkItems().then(setItems)
  },[])

  return (
    <div style={{padding:20}}>
      <h2>Work Items (click to open)</h2>
      <div style={{display:'flex', gap:20}}>
        <ul>
          {items.map(it => (
            <li key={it.id}>
              <button onClick={()=>setSelected(it.id)}>{it.title} — {it.state}</button>
            </li>
          ))}
        </ul>
        <div style={{flex:1}}>
          {selected ? <WorkItemView id={selected} /> : <div>Select a work item</div>}
        </div>
      </div>
    </div>
  )
}
```

## Frontend: src/WorkItemView.jsx

```jsx
import React, { useEffect, useState, useRef } from 'react'
import * as signalR from '@microsoft/signalr'
import { getWorkItem, updateWorkItemState } from './api'

export default function WorkItemView({ id }){
  const [item, setItem] = useState(null)
  const connectionRef = useRef(null)

  useEffect(()=>{
    let mounted = true

    getWorkItem(id).then(it => { if(mounted) setItem(it) })

    const conn = new signalR.HubConnectionBuilder()
      .withUrl('http://localhost:5000/hubs/workitems', { skipNegotiation: true, transport: signalR.HttpTransportType.WebSockets })
      .withAutomaticReconnect()
      .build()

    connectionRef.current = conn

    conn.on('WorkItemUpdated', updated => {
      if(+updated.id === +id) {
        setItem(updated)
      }
    })

    conn.start().then(()=>{
      // join the group for this workitem so we only receive relevant updates
      conn.invoke('JoinGroup', String(id)).catch(console.error)
    }).catch(console.error)

    return ()=>{
      mounted = false
      if (conn && conn.state === signalR.HubConnectionState.Connected){
        conn.invoke('LeaveGroup', String(id)).catch(console.error)
      }
      conn.stop().catch(()=>{})
    }
  },[id])

  if(!item) return <div>Loading...</div>

  const onChangeState = (newState) =>{
    updateWorkItemState(id, newState).then(updated => setItem(updated)).catch(console.error)
  }

  return (
    <div style={{border:'1px solid #ddd', padding:12}}>
      <h3>{item.title}</h3>
      <p>{item.description}</p>
      <p><strong>State:</strong> {item.state}</p>

      <div>
        <button onClick={()=>onChangeState('New')}>New</button>
        <button onClick={()=>onChangeState('Active')}>Active</button>
        <button onClick={()=>onChangeState('Resolved')}>Resolved</button>
        <button onClick={()=>onChangeState('Closed')}>Closed</button>
      </div>
    </div>
  )
}
```

## Frontend: src/api.js

```js
const API = 'http://localhost:5000/api/workitems'

export async function getAllWorkItems(){
  const res = await fetch(API)
  return await res.json()
}

export async function getWorkItem(id){
  const res = await fetch(`${API}/${id}`)
  return await res.json()
}

export async function updateWorkItemState(id, state){
  const res = await fetch(`${API}/${id}/state`, {
    method: 'PUT',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ state })
  })
  return await res.json()
}
```

## Frontend: src/index.css

```css
body { font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial; }
button { margin-right:8px }
```

---

## Notes & Tips

- This example uses an **in-memory** repository. In production you would use a database and likely optimistic concurrency or versioning.
- Grouping by work item id reduces traffic; only clients viewing a specific work item receive updates for it.
- SignalR connection options: in environments where WebSockets are not available, SignalR will fall back to other transports.
- Ensure CORS configuration allows credentials and origin between frontend and backend.
- For secure production deployments, use HTTPS, proper authentication (JWT/cookies), and restrict hub access.

---

If you want, I can:
- Convert the backend into a ready-to-run .NET solution with `.csproj` and `launchSettings.json`.
- Add authentication (JWT) and authorization scaffolding so only authorized users can update or subscribe.
- Provide a Dockerfile for both services.

Happy coding!
