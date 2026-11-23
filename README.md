-- Heavy On Ground, Light In Air + draggable weight menu (Roblox / Delta Executor)
-- Weight box value = part density applied to your own character while grounded.
-- Workspace.Gravity is never touched. Values marked TUNE are untested starting points.

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")

local CONFIG = {
    DefaultWeight = 0.7,     -- TUNE: Roblox default part density, assumed not read from your game
    MinWeight = 0.0001,      -- PhysicalProperties needs density > 0
    MaxWeight = 100000,      -- TUNE: upper clamp so a typo cannot freeze physics
    Friction = 0.3,
    Elasticity = 0,
    FrictionWeight = 1,
    ElasticityWeight = 1,
    LandDelay = 0.05,        -- seconds of stable ground contact before applying the weight again
    ToggleKey = Enum.KeyCode.H,
}

local player = Players.LocalPlayer

local state = {
    enabled = false,
    active = false,          -- true while the weight is currently applied
    weight = CONFIG.DefaultWeight,
    character = nil,
    humanoid = nil,
    originals = setmetatable({}, { __mode = "k" }),
    charConnections = {},
    globalConnections = {},
    landToken = 0,
}

local function clampWeight(value)
    if value ~= value then
        return nil
    end
    if value < CONFIG.MinWeight then
        return CONFIG.MinWeight
    end
    if value > CONFIG.MaxWeight then
        return CONFIG.MaxWeight
    end
    return value
end

local function collectParts(character)
    local parts = {}
    for _, descendant in ipairs(character:GetDescendants()) do
        if descendant:IsA("BasePart") then
            table.insert(parts, descendant)
        end
    end
    return parts
end

local function applyWeight()
    if not state.character then
        return
    end
    for _, part in ipairs(collectParts(state.character)) do
        if state.originals[part] == nil then
            state.originals[part] = part.CustomPhysicalProperties or false
        end
        part.CustomPhysicalProperties = PhysicalProperties.new(
            state.weight,
            CONFIG.Friction,
            CONFIG.Elasticity,
            CONFIG.FrictionWeight,
            CONFIG.ElasticityWeight
        )
    end
    state.active = true
end

local function removeWeight()
    state.landToken = state.landToken + 1
    if not state.active then
        return
    end
    for part, original in pairs(state.originals) do
        if part and part.Parent then
            if original == false then
                part.CustomPhysicalProperties = nil
            else
                part.CustomPhysicalProperties = original
            end
        end
    end
    state.originals = setmetatable({}, { __mode = "k" })
    state.active = false
end

local function isOnSolidGround(humanoid)
    local floor = humanoid.FloorMaterial
    return floor ~= Enum.Material.Air and floor ~= Enum.Material.Water
end

local function scheduleWeight()
    state.landToken = state.landToken + 1
    local token = state.landToken
    task.delay(CONFIG.LandDelay, function()
        if token ~= state.landToken or not state.enabled or not state.humanoid then
            return
        end
        if isOnSolidGround(state.humanoid) then
            applyWeight()
        end
    end)
end

local function onStateChanged(_, newState)
    if not state.enabled then
        return
    end
    if newState == Enum.HumanoidStateType.Jumping
        or newState == Enum.HumanoidStateType.Freefall then
        removeWeight()
    elseif newState == Enum.HumanoidStateType.Landed
        or newState == Enum.HumanoidStateType.Running
        or newState == Enum.HumanoidStateType.RunningNoPhysics then
        scheduleWeight()
    end
end

local function disconnectCharacter()
    for _, connection in ipairs(state.charConnections) do
        connection:Disconnect()
    end
    state.charConnections = {}
end

local function bindCharacter(character)
    disconnectCharacter()
    state.character = character
    state.humanoid = character:WaitForChild("Humanoid", 10)
    state.active = false
    state.originals = setmetatable({}, { __mode = "k" })
    if not state.humanoid then
        return
    end

    table.insert(state.charConnections,
        state.humanoid.StateChanged:Connect(onStateChanged))
    table.insert(state.charConnections,
        state.humanoid.Died:Connect(removeWeight))

    if state.enabled and isOnSolidGround(state.humanoid) then
        scheduleWeight()
    end
end

local function start()
    state.enabled = true
    if player.Character then
        bindCharacter(player.Character)
    end
end

local function stop()
    state.enabled = false
    removeWeight()
    disconnectCharacter()
end

-- Menu ---------------------------------------------------------------------

local function buildMenu()
    local gui = Instance.new("ScreenGui")
    gui.Name = "HeavyWeightMenu"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true

    local ok = pcall(function()
        gui.Parent = CoreGui
    end)
    if not ok then
        gui.Parent = player:WaitForChild("PlayerGui")
    end

    local frame = Instance.new("Frame")
    frame.Name = "Panel"
    frame.Size = UDim2.fromOffset(220, 120)
    frame.Position = UDim2.new(0.5, -110, 0.3, 0)
    frame.BackgroundColor3 = Color3.fromRGB(28, 28, 34)
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.Parent = gui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = frame

    local title = Instance.new("TextLabel")
    title.Name = "TitleBar"
    title.Size = UDim2.new(1, 0, 0, 28)
    title.BackgroundColor3 = Color3.fromRGB(45, 45, 56)
    title.BorderSizePixel = 0
    title.Text = "Weight  (drag here)"
    title.TextColor3 = Color3.fromRGB(235, 235, 240)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 14
    title.Parent = frame

    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 8)
    titleCorner.Parent = title

    local hint = Instance.new("TextLabel")
    hint.Size = UDim2.new(1, -16, 0, 18)
    hint.Position = UDim2.fromOffset(8, 34)
    hint.BackgroundTransparency = 1
    hint.Text = "Default " .. tostring(CONFIG.DefaultWeight) .. "  |  higher = heavier"
    hint.TextColor3 = Color3.fromRGB(170, 170, 185)
    hint.Font = Enum.Font.Gotham
    hint.TextSize = 12
    hint.TextXAlignment = Enum.TextXAlignment.Left
    hint.Parent = frame

    local box = Instance.new("TextBox")
    box.Name = "WeightBox"
    box.Size = UDim2.new(1, -16, 0, 30)
    box.Position = UDim2.fromOffset(8, 56)
    box.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
    box.BorderSizePixel = 0
    box.Text = tostring(CONFIG.DefaultWeight)
    box.PlaceholderText = "weight"
    box.TextColor3 = Color3.fromRGB(255, 255, 255)
    box.Font = Enum.Font.Code
    box.TextSize = 16
    box.ClearTextOnFocus = false
    box.Parent = frame

    local boxCorner = Instance.new("UICorner")
    boxCorner.CornerRadius = UDim.new(0, 6)
    boxCorner.Parent = box

    local status = Instance.new("TextLabel")
    status.Size = UDim2.new(1, -16, 0, 18)
    status.Position = UDim2.fromOffset(8, 92)
    status.BackgroundTransparency = 1
    status.Text = "Applied: " .. tostring(state.weight)
    status.TextColor3 = Color3.fromRGB(120, 220, 150)
    status.Font = Enum.Font.Gotham
    status.TextSize = 12
    status.TextXAlignment = Enum.TextXAlignment.Left
    status.Parent = frame

    box.FocusLost:Connect(function()
        local parsed = clampWeight(tonumber(box.Text))
        if parsed == nil then
            box.Text = tostring(state.weight)
            status.Text = "Invalid number, kept " .. tostring(state.weight)
            status.TextColor3 = Color3.fromRGB(240, 120, 120)
            return
        end
        state.weight = parsed
        box.Text = tostring(parsed)
        status.Text = "Applied: " .. tostring(parsed)
        status.TextColor3 = Color3.fromRGB(120, 220, 150)
        if state.enabled and state.active then
            removeWeight()
            applyWeight()
        end
    end)

    local dragging = false
    local dragStart = Vector2.zero
    local startPosition = frame.Position

    title.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = Vector2.new(input.Position.X, input.Position.Y)
            startPosition = frame.Position
        end
    end)

    table.insert(state.globalConnections,
        UserInputService.InputChanged:Connect(function(input)
            if not dragging then
                return
            end
            if input.UserInputType == Enum.UserInputType.MouseMovement
                or input.UserInputType == Enum.UserInputType.Touch then
                local delta = Vector2.new(input.Position.X, input.Position.Y) - dragStart
                frame.Position = UDim2.new(
                    startPosition.X.Scale,
                    startPosition.X.Offset + delta.X,
                    startPosition.Y.Scale,
                    startPosition.Y.Offset + delta.Y
                )
            end
        end))

    table.insert(state.globalConnections,
        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end))

    return gui
end

local menu = buildMenu()

table.insert(state.globalConnections, player.CharacterAdded:Connect(bindCharacter))

table.insert(state.globalConnections,
    UserInputService.InputBegan:Connect(function(input, processed)
        if processed or input.KeyCode ~= CONFIG.ToggleKey then
            return
        end
        if state.enabled then
            stop()
        else
            start()
        end
    end))

start()
