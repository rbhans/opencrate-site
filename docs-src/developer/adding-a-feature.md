# Adding a Feature

Most new features in OpenCrate follow the same implementation path. This guide walks through each step with concrete file paths and wiring instructions.

## Step-by-step

### 1. Create the store

If your feature needs persistent state, start with a store.

**Create the file:** `src/store/my_feature_store.rs`

Follow the standard store pattern (see [Developer Guide](index.md#the-store-pattern)):

1. Define a command enum with all operations, each carrying a `oneshot::Sender` for the reply.
2. Define your public struct with an `mpsc::Sender<Cmd>` field.
3. In the constructor, spawn a `std::thread` that opens a SQLite connection at `data/my_feature.db` (WAL mode), runs migrations, and loops on `rx.blocking_recv()`.
4. Accept `Option<EventBus>` in the constructor. Publish relevant events after successful mutations. Use `None` in tests.
5. Expose async methods that send commands and await oneshot replies.

**Register the module:** Add `pub mod my_feature_store;` to `src/store/mod.rs`.

### 2. Wire into the platform

Open `src/platform.rs` and add your store to the initialization sequence.

1. **Create the store** in `init_platform()`:
   ```rust
   let my_feature_store = MyFeatureStore::new(
       &data_dir.join("my_feature.db").to_string_lossy(),
       Some(event_bus.clone()),
   );
   ```

2. **Add to `AutomationState`** (or `ModelState` if it is core model data):
   ```rust
   pub struct AutomationState {
       // ... existing fields
       pub my_feature_store: MyFeatureStore,
   }
   ```

3. **Add to `SharedPlatform`** if the API needs access:
   ```rust
   pub struct SharedPlatform {
       // ... existing fields
       pub my_feature_store: MyFeatureStore,
   }
   ```

### 3. Add API routes

If your feature needs HTTP endpoints, create a route module.

**Create the file:** `src/api/routes/my_feature.rs`

```rust
use axum::{extract::State, Json, Router, routing::{get, post}};
use crate::api::ApiState;

pub fn routes() -> Router<ApiState> {
    Router::new()
        .route("/api/my-feature", get(list_items).post(create_item))
        .route("/api/my-feature/:id", get(get_item).put(update_item).delete(delete_item))
}

async fn list_items(State(state): State<ApiState>) -> Json<Vec<MyItem>> {
    let items = state.my_feature_store.list().await;
    Json(items)
}

// ... other handlers
```

**Register the routes:** In `src/api/routes/mod.rs`, declare the module and merge the router:

```rust
pub mod my_feature;

pub fn all_routes() -> Router<ApiState> {
    Router::new()
        // ... existing routes
        .merge(my_feature::routes())
}
```

**Add to `ApiState`:** In `src/api/mod.rs`, add your store to the `ApiState` struct and populate it from `SharedPlatform`.

### 4. Add permissions

If your feature needs access control, add a permission variant.

**In `src/auth.rs`:**

```rust
pub enum Permission {
    // ... existing variants
    ManageMyFeature,
}
```

Gate your API handlers with the permission check. Add audit action variants for mutations (Create, Update, Delete operations).

### 5. Add GUI components

If your feature needs a user interface, create Dioxus components.

**Create the file:** `src/gui/components/my_feature_view.rs`

```rust
use dioxus::prelude::*;
use crate::gui::state::AppState;

#[component]
pub fn MyFeatureView(cx: Scope) -> Element {
    let state = use_shared_state::<AppState>(cx)?;
    // Component implementation
}
```

**Register the component:** Add `pub mod my_feature_view;` to `src/gui/components/mod.rs`.

**Add to navigation:** Wire the component into the appropriate tab in the GUI. Most features go under the Config section as a new tab, or under an existing view as a sub-tab.

**Add to `AppState`:** If your component needs state beyond what the store provides, add fields to `src/gui/state.rs`.

### 6. Add event subscribers

If your feature needs to react to system events, create an EventBus subscriber.

```rust
pub struct MyFeatureEngine {
    // ...
}

impl MyFeatureEngine {
    pub fn start(event_bus: &EventBus, store: MyFeatureStore) {
        let mut rx = event_bus.subscribe();
        tokio::spawn(async move {
            while let Ok(event) = rx.recv().await {
                match event.as_ref() {
                    Event::AlarmRaised { .. } => {
                        // React to alarms
                    }
                    Event::ValueChanged { .. } => {
                        // React to value changes
                    }
                    _ => {}
                }
            }
        });
    }
}
```

Start the engine in `init_platform()` after creating all stores:

```rust
MyFeatureEngine::start(&event_bus, my_feature_store.clone());
```

### 7. Add to `lib.rs`

Declare your top-level module in `src/lib.rs` if you created a new module directory:

```rust
pub mod my_feature;
```

## Checklist

Use this as a quick reference when adding a feature:

- [ ] Store: `src/store/my_feature_store.rs` + register in `src/store/mod.rs`
- [ ] Platform: wire store in `src/platform.rs` (create instance, add to `AutomationState` / `SharedPlatform`)
- [ ] API routes: `src/api/routes/my_feature.rs` + register in `src/api/routes/mod.rs` + add store to `ApiState`
- [ ] Auth: permission variant in `src/auth.rs` + audit action variants
- [ ] GUI component: `src/gui/components/my_feature_view.rs` + register in `src/gui/components/mod.rs`
- [ ] GUI state: add fields to `src/gui/state.rs` if needed
- [ ] Event subscriber: create engine/subscriber + start in `init_platform()`
- [ ] Module declaration: add `pub mod` to `src/lib.rs` if new top-level module
- [ ] Tests: store tests with `None` EventBus and `:memory:` database

## Examples from the codebase

These existing features followed exactly this pattern and serve as good references:

| Feature | Store | API routes | GUI component | Event subscriber |
|---------|-------|------------|---------------|------------------|
| **MQTT** | `mqtt_store.rs` | (in routes/mod.rs) | Config > MQTT tab | `MqttPublisher` |
| **Webhooks** | `webhook_store.rs` | `routes/webhooks.rs` | `webhook_settings.rs` | `WebhookDispatcher` |
| **Reporting** | `report_store.rs` | `routes/` (reports) | Config > Reports tab | Report scheduler |
| **Energy** | `energy_store.rs` | `routes/energy.rs` | `energy_view.rs` | Energy scheduler |
| **Commissioning** | `commissioning_store.rs` | -- | `commissioning_tab.rs` | -- |
| **Notifications** | `notification_store.rs` | -- | Config > Alarm Routing tab | `AlarmRouter` |

Each of these follows the same store-then-platform-then-API-then-GUI progression described above.
