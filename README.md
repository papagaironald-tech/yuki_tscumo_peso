-- Heavy On Ground, Light In Air (Roblox / Delta Executor)
-- Heavy while grounded (sinks boats), normal mass while jumping or falling.
-- Only your own character's parts are changed. Workspace.Gravity is never touched.
-- Values marked TUNE are untested starting points.

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")

local CONFIG = {
    HeavyDensity = 60,       -- TUNE: density while grounded. Roblox default part density is about 0.7.
    Friction = 0.3,
    Elasticity = 0,
    FrictionWeight = 1,
    ElasticityWeight = 1,
    LandDelay = 0.05,        -- seconds of stable ground contact before turning heavy again
    ToggleKey = Enum.KeyCode.H,
}

local player = Players.LocalPlayer

local state = {
    enabled = false,
    heavy = false,
    character = nil,
    humanoid = nil,
    originals = setmetatable({}, { __mode = "k" }),
    connections = {},
    landToken = 0,
}

local function collectParts(character)
    local parts = {}
    for _, descendant in ipairs(character:GetDescendants()) do
        if descendant:IsA("BasePart") then
            table.insert(parts, descendant)
        end
    end
    return parts
end

local function setHeavy()
    if not state.character or state.heavy then
        return
    end
    for _, part in ipairs(collectParts(state.character)) do
        if state.originals[part] == nil then
            state.originals[part] = part.CustomPhysicalProperties or false
        end
        part.CustomPhysicalProperties = PhysicalProperties.new(
            CONFIG.HeavyDensity,
            CONFIG.Friction,
            CONFIG.Elasticity,
            CONFIG.FrictionWeight,
            CONFIG.ElasticityWeight
        )
    end
    state.heavy = true
end

local function setLight()
    state.landToken = state.landToken + 1
    if not state.heavy then
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
    state.heavy = false
end

local function isOnSolidGround(humanoid)
    if humanoid.FloorMaterial == Enum.Material.Air then
        return false
    end
    if humanoid.FloorMaterial == Enum.Material.Water then
        return false
    end
    return true
end

local function scheduleHeavy()
    state.landToken = state.landToken + 1
    local token = state.landToken
    task.delay(CONFIG.LandDelay, function()
        if token ~= state.landToken or not state.enabled or not state.humanoid then
            return
        end
        if isOnSolidGround(state.humanoid) then
            setHeavy()
        end
    end)
end

local function onStateChanged(_, newState)
    if not state.enabled then
        return
    end
    if newState == Enum.HumanoidStateType.Jumping
        or newState == Enum.HumanoidStateType.Freefall then
        setLight()
    elseif newState == Enum.HumanoidStateType.Landed
        or newState == Enum.HumanoidStateType.Running
        or newState == Enum.HumanoidStateType.RunningNoPhysics then
        scheduleHeavy()
    end
end

local function disconnectCharacter()
    for _, connection in ipairs(state.connections) do
        connection:Disconnect()
    end
    state.connections = {}
end

local function bindCharacter(character)
    disconnectCharacter()
    state.character = character
    state.humanoid = character:WaitForChild("Humanoid", 10)
    state.heavy = false
    state.originals = setmetatable({}, { __mode = "k" })
    if not state.humanoid then
        return
    end

    table.insert(state.connections,
        state.humanoid.StateChanged:Connect(onStateChanged))

    table.insert(state.connections,
        state.humanoid.Died:Connect(function()
            setLight()
        end))

    if state.enabled and isOnSolidGround(state.humanoid) then
        scheduleHeavy()
    end
end

local function start()
    state.enabled = true
    if player.Character then
        bindCharacter(player.Character)
    end
    table.insert(state.connections, player.CharacterAdded:Connect(bindCharacter))
end

local function stop()
    state.enabled = false
    setLight()
    disconnectCharacter()
end

UserInputService.InputBegan:Connect(function(input, processed)
    if processed or input.KeyCode ~= CONFIG.ToggleKey then
        return
    end
    if state.enabled then
        stop()
    else
        start()
    end
end)

start()
