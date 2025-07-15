local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local hrp = character:WaitForChild("HumanoidRootPart")

local flying = false
local speed = 50
local fixedHeight = 60
local control = {F = 0, B = 0, L = 0, R = 0}

-- Ponto de spawn original do personagem
local spawnPoint = Workspace:FindFirstChildOfClass("SpawnLocation")
if spawnPoint then
    spawnPoint = spawnPoint.Position
else
    spawnPoint = hrp.Position -- fallback
end

-- Interface botão
local screenGui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
screenGui.Name = "FlyUI"
screenGui.ResetOnSpawn = false

local toggleButton = Instance.new("TextButton")
toggleButton.Size = UDim2.new(0, 100, 0, 40)
toggleButton.Position = UDim2.new(1, -110, 0.85, 0)
toggleButton.AnchorPoint = Vector2.new(0, 0)
toggleButton.BackgroundColor3 = Color3.fromRGB(200, 30, 30)
toggleButton.Text = "Fly: OFF"
toggleButton.TextScaled = true
toggleButton.Font = Enum.Font.SourceSansBold
toggleButton.TextColor3 = Color3.new(1, 1, 1)
toggleButton.Parent = screenGui

-- Função para mover para spawn após subir
local function moveToSpawn()
    local arrived = false

    local connection
    connection = RunService.RenderStepped:Connect(function()
        if not flying then
            connection:Disconnect()
            return
        end

        local directionToSpawn = (spawnPoint - hrp.Position)
        local distance = directionToSpawn.Magnitude

        -- Se chegou perto do spawn, para o voo
        if distance <= 5 then
            flying = false
            humanoid.PlatformStand = false
            toggleButton.Text = "Fly: OFF"
            toggleButton.BackgroundColor3 = Color3.fromRGB(200, 30, 30)
            connection:Disconnect()
            return
        end

        -- Move em direção ao spawn em velocidade controlada
        local horizontalDir = Vector3.new(directionToSpawn.X, 0, directionToSpawn.Z).Unit * speed

        -- Mantém altura fixa (fixedHeight acima do chão)
        local rayOrigin = hrp.Position
        local rayDirection = Vector3.new(0, -200, 0)
        local raycastParams = RaycastParams.new()
        raycastParams.FilterDescendantsInstances = {character}
        raycastParams.FilterType = Enum.RaycastFilterType.Blacklist

        local raycastResult = Workspace:Raycast(rayOrigin, rayDirection, raycastParams)
        local groundY = raycastResult and raycastResult.Position.Y or 0
        local desiredY = math.max(groundY + fixedHeight, 10)

        local heightDifference = desiredY - hrp.Position.Y
        local verticalSpeed = math.clamp(heightDifference * 2, -100, 100)
        local vertical = Vector3.new(0, verticalSpeed, 0)

        humanoid:Move(horizontalDir + vertical, true)
    end)
end

-- Função de voo inicial (subir até fixedHeight)
local function fly()
    if flying then return end
    flying = true
    humanoid.PlatformStand = true

    -- Eleva o personagem se estiver no chão
    if hrp.Position.Y < 5 then
        hrp.CFrame = hrp.CFrame + Vector3.new(0, 10, 0)
    end

    -- Subir até altura fixa
    local connection
    connection = RunService.RenderStepped:Connect(function()
        if not flying then
            connection:Disconnect()
            return
        end

        local rayOrigin = hrp.Position
        local rayDirection = Vector3.new(0, -200, 0)
        local raycastParams = RaycastParams.new()
        raycastParams.FilterDescendantsInstances = {character}
        raycastParams.FilterType = Enum.RaycastFilterType.Blacklist

        local raycastResult = Workspace:Raycast(rayOrigin, rayDirection, raycastParams)
        local groundY = raycastResult and raycastResult.Position.Y or 0
        local desiredY = math.max(groundY + fixedHeight, 10)

        local heightDifference = desiredY - hrp.Position.Y
        local verticalSpeed = math.clamp(heightDifference * 2, -100, 100)
        local vertical = Vector3.new(0, verticalSpeed, 0)

        -- Movimento vertical só (sem horizontal)
        humanoid:Move(vertical, true)

        -- Quando chegar perto da altura desejada, para essa fase e começa o movimento para spawn
        if math.abs(heightDifference) <= 1 then
            connection:Disconnect()
            moveToSpawn()
        end
    end)
end

-- Parar voo
local function stopFlying()
    flying = false
    humanoid.PlatformStand = false
end

-- Botão liga/desliga voo
toggleButton.MouseButton1Click:Connect(function()
    if flying then
        stopFlying()
        toggleButton.Text = "Fly: OFF"
        toggleButton.BackgroundColor3 = Color3.fromRGB(200, 30, 30)
    else
        fly()
        toggleButton.Text = "Fly: ON"
        toggleButton.BackgroundColor3 = Color3.fromRGB(30, 200, 30)
    end
end)

-- Controles de direção (WASD)
UIS.InputBegan:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.W then control.F = -1 end
    if input.KeyCode == Enum.KeyCode.S then control.B = 1 end
    if input.KeyCode == Enum.KeyCode.A then control.L = -1 end
    if input.KeyCode == Enum.KeyCode.D then control.R = 1 end
end)

UIS.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.W then control.F = 0 end
    if input.KeyCode == Enum.KeyCode.S then control.B = 0 end
    if input.KeyCode == Enum.KeyCode.A then control.L = 0 end
    if input.KeyCode == Enum.KeyCode.D then control.R = 0 end
end)
