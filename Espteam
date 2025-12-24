-- Inventory under-feet ESP (LocalScript)
-- Показывает под ногами у других игроков:
-- 1) список предметов в Backpack
-- 2) что у них в руках (equipped tool)
-- Без подсветки, только текст.

local REFRESH_INTERVAL = 1      -- частота проверки на случай, если события не сработают
local TEXT_OFFSET = Vector3.new(0, -3, 0) -- смещение под ногами (отрегулируй если нужно)
local MAX_LINES = 20            -- максимум строк в UI (предотвращает переполнение)

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

local ESPs = {} -- таблица: player -> {Billboard, TextLabel, conns = {}}

local function safeWait(child, name, timeout)
    if child:FindFirstChild(name) then return child[name] end
    return child:WaitForChild(name, timeout)
end

local function buildTextForPlayer(player)
    -- Что в руке (в Character)
    local inHand = "В руках: Нет"
    if player.Character then
        for _, obj in pairs(player.Character:GetChildren()) do
            if obj:IsA("Tool") then
                inHand = "В руках: " .. obj.Name
                break
            end
        end
    end

    -- Список из Backpack
    local items = {}
    local bp = player:FindFirstChild("Backpack")
    if bp then
        for _, obj in ipairs(bp:GetChildren()) do
            table.insert(items, obj.Name)
            if #items >= MAX_LINES - 2 then break end
        end
    end

    if #items == 0 then
        items[1] = "Backpack: Пусто"
    else
        table.insert(items, 1, "Backpack:")
    end

    table.insert(items, 1, inHand)
    return table.concat(items, "\n")
end

local function createESP(player)
    if ESPs[player] or player == LocalPlayer then return end

    local playerGui = LocalPlayer:WaitForChild("PlayerGui")

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "InventoryESP_Under"
    billboard.Size = UDim2.new(0, 220, 0, 100)
    billboard.StudsOffset = TEXT_OFFSET
    billboard.AlwaysOnTop = true
    billboard.ResetOnSpawn = false
    billboard.MaxDistance = 250
    billboard.Parent = playerGui

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, -6, 1, -6)
    textLabel.Position = UDim2.new(0, 3, 0, 3)
    textLabel.BackgroundTransparency = 1
    textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    textLabel.TextWrapped = true
    textLabel.TextScaled = true
    textLabel.Font = Enum.Font.SourceSans
    textLabel.Text = ""
    textLabel.TextYAlignment = Enum.TextYAlignment.Top
    textLabel.Parent = billboard

    ESPs[player] = {Billboard = billboard, TextLabel = textLabel, conns = {}}

    -- Функция обновления и установка Adornee
    local function attachAdornee(char)
        if not char then
            billboard.Adornee = nil
            return
        end
        local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
        billboard.Adornee = root
    end

    -- Подключаемся к CharacterAdded чтобы сразу повесить Adornee
    ESPs[player].conns.CharacterAdded = player.CharacterAdded:Connect(function(char)
        attachAdornee(char)
    end)
    if player.Character then
        attachAdornee(player.Character)
    end

    -- Обновление при изменениях в Backpack
    local function connectBackpackEvents()
        local bp = player:FindFirstChild("Backpack") or player:WaitForChild("Backpack", 1)
        if not bp then return end
        -- подключаем удаление/добавление
        ESPs[player].conns.BPChildAdded = bp.ChildAdded:Connect(function() ESPs[player].TextLabel.Text = buildTextForPlayer(player) end)
        ESPs[player].conns.BPChildRemoved = bp.ChildRemoved:Connect(function() ESPs[player].TextLabel.Text = buildTextForPlayer(player) end)
    end

    -- Обновление при смене Tool в Character
    local function connectCharacterToolEvents()
        if not player.Character then return end
        ESPs[player].conns.CharChildAdded = player.Character.ChildAdded:Connect(function(child)
            if child:IsA("Tool") then ESPs[player].TextLabel.Text = buildTextForPlayer(player) end
        end)
        ESPs[player].conns.CharChildRemoved = player.Character.ChildRemoved:Connect(function(child)
            if child:IsA("Tool") then ESPs[player].TextLabel.Text = buildTextForPlayer(player) end
        end)
    end

    -- Следим за появлением Backpack/Character
    ESPs[player].conns.BackpackChanged = player:GetPropertyChangedSignal("Backpack"):Connect(function()
        -- сброс предыдущих BP-коннеков если были
        if ESPs[player].conns.BPChildAdded then ESPs[player].conns.BPChildAdded:Disconnect() end
        if ESPs[player].conns.BPChildRemoved then ESPs[player].conns.BPChildRemoved:Disconnect() end
        connectBackpackEvents()
        ESPs[player].TextLabel.Text = buildTextForPlayer(player)
    end)

    ESPs[player].conns.CharacterAdded_Internal = player.CharacterAdded:Connect(function(char)
        -- отключаем старые Character-соединения если есть
        if ESPs[player].conns.CharChildAdded then ESPs[player].conns.CharChildAdded:Disconnect() end
        if ESPs[player].conns.CharChildRemoved then ESPs[player].conns.CharChildRemoved:Disconnect() end
        attachAdornee(char)
        connectCharacterToolEvents()
        ESPs[player].TextLabel.Text = buildTextForPlayer(player)
    end)

    -- Попытка подключить сразу (если уже есть объекты)
    connectBackpackEvents()
    connectCharacterToolEvents()

    -- Первичное заполнение текста
    ESPs[player].TextLabel.Text = buildTextForPlayer(player)
end

local function destroyESP(player)
    local data = ESPs[player]
    if not data then return end
    for k, conn in pairs(data.conns) do
        if conn and typeof(conn.Disconnect) == "function" then
            pcall(function() conn:Disconnect() end)
        end
    end
    if data.Billboard and data.Billboard.Parent then
        data.Billboard:Destroy()
    end
    ESPs[player] = nil
end

-- Инициализация для уже в игре игроков
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        createESP(player)
    end
end

-- События появления/ухода игроков
Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then
        createESP(player)
    end
end)
Players.PlayerRemoving:Connect(function(player)
    destroyESP(player)
end)

-- Резервное периодическое обновление (на случай, если события пропустили)
spawn(function()
    while true do
        task.wait(REFRESH_INTERVAL)
        for player, data in pairs(ESPs) do
            if player and data and data.TextLabel then
                data.TextLabel.Text = buildTextForPlayer(player)
                -- если Adornee потерялся — попытаться восстановить
                if not data.Billboard.Adornee and player.Character then
                    local root = player.Character:FindFirstChild("HumanoidRootPart") or player.Character:FindFirstChild("Torso") or player.Character:FindFirstChild("UpperTorso")
                    data.Billboard.Adornee = root
                end
            end
        end
    end
end)
