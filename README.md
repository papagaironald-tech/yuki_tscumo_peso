-- Heavy Character (Roblox / Delta Executor)
-- Raises the mass of your own character's parts without changing gravity or jump.
-- Source of each value: CONFIG below. Values marked TUNE are untested starting points.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local CONFIG = {
    -- Density of each part. Roblox default for a part is about 0.7. TUNE.
    Density = 100,
    Friction = 0.3,
    Elasticity = 0,
    FrictionWeight = 1,
    ElasticityWeight = 1,
    -- Keep jump height the same by scaling JumpPower/JumpHeight with the effect of mass.
    -- Humanoid jumps apply their own velocity, so this is normally 1 (no change). TUNE.
    JumpScale = 1,
    -- Re-apply every second in case the game resets properties.
    ReapplyInterval = 1,
    ToggleKey = Enum.KeyCode.H,
}

local player = Players.LocalPlayer
local enabled = false
local originals = setmetatable({}, { __mode = "k" })
local connections = {}
local lastApply = 0

local function characterParts(character)
    local parts = {}
    for _, descendant in ipairs(character:GetDescendants()) do
        if descendant:IsA("BasePart") then
            table.insert(parts, descendant)
        end
    end
    return parts
end

local function applyHeavy(character)
    for _, part in ipairs(characterParts(character)) do
        if originals[part] == nil then
            originals[part] = part.CustomPhysicalProperties or false
        end
        part.CustomPhysicalProperties = PhysicalProperties.new(
            CONFIG.Density,
            CONFIG.Friction,
            CONFIG.Elasticity,
            CONFIG.FrictionWeight,
            CONFIG.ElasticityWeight
        )
    end

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid and CONFIG.JumpScale ~= 1 then
        if humanoid.UseJumpPower then
            humanoid.JumpPower = humanoid.JumpPower * CONFIG.JumpScale
        else
            humanoid.JumpHeight = humanoid.JumpHeight * CONFIG.JumpScale
        end
    end
end

local function restore()
    for part, original in pairs(originals) do
        if part and part.Parent then
            if original == false then
                part.CustomPhysicalProperties = nil
            else
                part.CustomPhysicalProperties = original
            end
        end
    end
    originals = setmetatable({}, { __mode = "k" })
end

local function stop()
    enabled = false
    for _, connection in ipairs(connections) do
        connection:Disconnect()
    end
    connections = {}
    restore()
end

local function start()
    enabled = true
    local character = player.Character
    if character then
        applyHeavy(character)
    end

    table.insert(connections, player.CharacterAdded:Connect(function(newCharacter)
        newCharacter:WaitForChild("HumanoidRootPart")
        if enabled then
            applyHeavy(newCharacter)
        end
    end))

    table.insert(connections, RunService.Heartbeat:Connect(function()
        if not enabled then
            return
        end
        local now = os.clock()
        if now - lastApply < CONFIG.ReapplyInterval then
            return
        end
        lastApply = now
        local current = player.Character
        if current then
            applyHeavy(current)
        end
    end))
end

table.insert(connections, UserInputService.InputBegan:Connect(function(input, processed)
    if processed or input.KeyCode ~= CONFIG.ToggleKey then
        return
    end
    if enabled then
        stop()
    else
        start()
    end
end))

start()
