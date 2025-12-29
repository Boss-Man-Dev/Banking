# Bachelor Bank - FiveM Integration Guide

This project is built with React + Vite. To use it in your FiveM server, follow these steps.

## 2. Creating the Resource

1.  Go to your FiveM server's `resources` directory.
2.  Create a new folder, e.g., `bachelor-bank`.
3.  Inside `bachelor-bank`, create a file named `fxmanifest.lua`.
4.  Create a folder named `html` inside `bachelor-bank`.
5.  **Copy the contents** of the `dist/public` folder (from step 1) into the `html` folder.

Your structure should look like this:
```
resources/
└── bachelor-bank/
    ├── fxmanifest.lua
    └── html/
        ├── index.html
        └── assets/
            ├── index-xxxx.js
            └── index-xxxx.css
```

## 3. Configuration (fxmanifest.lua)

Paste the following into your `fxmanifest.lua`:

```lua
fx_version 'cerulean'
game 'gta5'

ui_page 'html/index.html'

files {
    'html/index.html',
    'html/assets/*.js',
    'html/assets/*.css',
    'html/assets/*.png', -- if you have images
    'html/vite.svg'      -- if applicable
}
```

## 4. Lua Backend Integration (server/client)

You will need to create client and server scripts to handle the banking logic.

### Opening Specific Interfaces

The UI supports opening directly to the Bank or ATM interface.

**Open Bank:**
```lua
SendNUIMessage({
    action = "setVisible",
    data = {
        visible = true,
        path = "/bank"
    }
})
```

**Open ATM:**
```lua
SendNUIMessage({
    action = "setVisible",
    data = {
        visible = true,
        path = "/atm"
    }
})
```

**Admin Panel Access:**
The Admin Panel is accessible via the `F9` key if the user has `isAdmin: true` in their user data.

### Client Script (`client.lua`)

Create a `client.lua` file in your resource folder and add it to `fxmanifest.lua` (`client_script 'client.lua'`).

```lua
local display = false

-- Command to open the bank (for testing)
RegisterCommand("bank", function()
    SetDisplay(not display)
end)

-- Callback from React to Close UI
RegisterNUICallback("closeBankUI", function(data, cb)
    SetDisplay(false)
    cb('ok')
end)

-- Callback for Transactions
RegisterNUICallback("processTransaction", function(data, cb)
    -- data.type ('deposit', 'withdrawal', 'transfer', 'admin')
    -- data.amount
    -- data.process ('add_money', 'remove_money', 'delete_user') -- For admin actions
    
    -- Trigger server event to check money and update DB
    TriggerServerEvent('bachelor-bank:handleTransaction', data)
    
    -- Return true to UI to simulate success
    -- In a real app, you might want to wait for server response
    cb(true) 
end)

function SetDisplay(bool)
    display = bool
    SetNuiFocus(bool, bool)
    SendNUIMessage({
        action = "setVisible",
        data = {
            visible = bool,
            path = "/bank" -- Default path
        }
    })
end
```

### React NUI Handling

The React app listens for specific `message` events to update its state.

#### 1. updateUser
Updates the current user's profile and settings.

```lua
SendNUIMessage({
    action = "updateUser",
    data = {
        name = { first = "John", last = "Doe" },
        email = "john.doe@example.com",
        phone = "555-0123",
        balance = 50000,
        avatar = "https://url-to-avatar",
        isAdmin = true, -- Grants access to F9 Admin Panel
        notifications = {
            deposit = true,
            withdrawal = true,
            transferSent = true,
            transferReceived = true,
            atm = true,
            security = true,
            promos = false
        }
    }
})
```

#### 2. updateContacts (Also populates Admin User List)
This event serves two purposes:
1.  Populates the "Quick Transfer" contact list.
2.  Populates the "Users" list in the Admin Panel.

```lua
SendNUIMessage({
    action = "updateContacts",
    data = {
        {
            user = {
                accountId = "ACC-12345",
                name = { first = "Jane", last = "Smith" },
                email = "jane.smith@example.com",
                phone = "555-5678",
                avatar = "https://...",
                isAdmin = false
            },
            balance = 25000
        },
        -- Add more users...
    }
})
```

## 5. UI Features & Logic

### Notifications
The UI now includes a robust notification system:
-   **Popups (Toast):** Critical actions (transfers, deposits, admin changes) trigger a visual popup that auto-closes after 3 seconds.
-   **Persistent History:** These events are also logged in the "Notifications" bell menu.
-   **Admin Sync:** Admin actions (adding/removing money, deleting users) automatically generate persistent notifications.

### Transaction History
-   **Pagination:** Displays the last 25 transactions by default.
-   **Load More:** "..." button loads the next 25 transactions.
-   **Show Less:** Option to collapse the list back to 25 items.

### Admin Panel
-   **Access:** Press `F9` (requires `isAdmin: true`).
-   **Features:**
    -   View all users.
    -   Edit user details (Name, Email, Phone).
    -   Manage Wallet (Add/Remove money).
    -   Delete Users.
    -   Toggle Permissions (Admin access).
    -   Global Settings (App Name, Colors).

### Auto-Close Modals
-   Deposit, Withdrawal, and Transfer modals automatically close 3 seconds after a successful transaction.
-   Users can cancel this by clicking "Make Another Transaction" or "Send Another".

## 6. Important Notes

-   **Base Path**: The `vite.config.ts` has been updated to use `base: './'` to ensure assets load correctly in the NUI environment.
-   **Images**: If you use local images, ensure they are in the `public` folder and included in `fxmanifest.lua` files list.
