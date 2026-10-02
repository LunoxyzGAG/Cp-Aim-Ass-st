
-- DHEN AIM ASSIST + GOOD GRAPHICS
-- LocalScript
-- StarterPlayer > StarterPlayerScripts
-- Draggable GUI: PC + Mobile

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- AIM SETTINGS
local FOV_RADIUS = 100
local AIM_STRENGTH = 0.12
local MAX_DISTANCE = 300

local aiming = true
local currentTarget = nil
local switchRequested = false

-- GOOD GRAPHICS
Lighting.GlobalShadows = true
Lighting.Brightness = 2
Lighting.ExposureCompensation = 0.05
Lighting.EnvironmentDiffuseScale = 0.85
Lighting.EnvironmentSpecularScale = 0.8
Lighting.ClockTime = 14
Lighting.FogStart = 0
Lighting.FogEnd = 100000

local color = Lighting:FindFirstChild("DhenColor")
if not color then
    color = Instance.new("ColorCorrectionEffect")
    color.Name = "DhenColor"
    color.Parent = Lighting
end
color.Brightness = 0.02
color.Contrast = 0.10
color.Saturation = 0.06

local bloom = Lighting:FindFirstChild("DhenBloom")
if not bloom then
    bloom = Instance.new("BloomEffect")
    bloom.Name = "DhenBloom"
    bloom.Parent = Lighting
end
bloom.Intensity = 0.08
bloom.Size = 16
bloom.Threshold = 1.2

local sun = Lighting:FindFirstChild("DhenSunRays")
if not sun then
    sun = Instance.new("SunRaysEffect")
    sun.Name = "DhenSunRays"
    sun.Parent = Lighting
end
sun.Intensity = 0.025
sun.Spread = 0.8

-- SMALL GUI
local gui = Instance.new("ScreenGui")
gui.Name = "DhenAimMenu"
gui.ResetOnSpawn = false
gui.Parent = player:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Name = "Main"
frame.Size = UDim2.fromOffset(125, 42)
frame.Position = UDim2.new(1, -135, 0.5, -21)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
frame.BorderSizePixel = 0
frame.Active = true
frame.Parent = gui

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 7)
frameCorner.Parent = frame

local toggle = Instance.new("TextButton")
toggle.Size = UDim2.new(1, -8, 1, -8)
toggle.Position = UDim2.fromOffset(4, 4)
toggle.BackgroundColor3 = Color3.fromRGB(35, 160, 80)
toggle.TextColor3 = Color3.new(1, 1, 1)
toggle.Font = Enum.Font.GothamBold
toggle.TextSize = 11
toggle.Text = "AIM: ON"
toggle.Parent = frame

local buttonCorner = Instance.new("UICorner")
buttonCorner.CornerRadius = UDim.new(0, 5)
buttonCorner.Parent = toggle

-- AIM ON/OFF
toggle.Activated:Connect(function()
    aiming = not aiming

    if not aiming then
        currentTarget = nil
        switchRequested = false
    end

    toggle.Text = aiming and "AIM: ON" or "AIM: OFF"
    toggle.BackgroundColor3 = aiming
        and Color3.fromRGB(35, 160, 80)
        or Color3.fromRGB(170, 50, 50)
end)

-- DRAG GUI: PC + TOUCH
local dragging = false
local dragStart
local startPosition
local activeTouch
local dragMoved = false

local function updateDrag(input)
    local delta = input.Position - dragStart
    frame.Position = UDim2.new(
        startPosition.X.Scale,
        startPosition.X.Offset + delta.X,
        startPosition.Y.Scale,
        startPosition.Y.Offset + delta.Y
    )
end

frame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        dragMoved = false
        dragStart = input.Position
        startPosition = frame.Position

        if input.UserInputType == Enum.UserInputType.Touch then
            activeTouch = input
        end
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not dragging then return end

    local isMouseMove =
        input.UserInputType == Enum.UserInputType.MouseMovement
    local isActiveTouch =
        activeTouch and input == activeTouch

    if isMouseMove or isActiveTouch then
        if (input.Position - dragStart).Magnitude > 4 then
            dragMoved = true
            updateDrag(input)
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or (activeTouch and input == activeTouch) then
        dragging = false
        activeTouch = nil
    end
end)

-- SWITCH TARGET: Y KEY
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end

    if input.KeyCode == Enum.KeyCode.Y then
        switchRequested = true
    end
end)

-- WALL CHECK
local function canSeeTarget(character, part)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {
        player.Character,
        camera
    }

    local origin = camera.CFrame.Position
    local result = workspace:Raycast(
        origin,
        part.Position - origin,
        params
    )

    return result == nil or result.Instance:IsDescendantOf(character)
end

-- FIND TARGETS
local function getTargets()
    local targets = {}
    local center = camera.ViewportSize / 2

    for _, other in ipairs(Players:GetPlayers()) do
        if other == player then continue end

        local character = other.Character
        if not character then continue end

        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local head = character:FindFirstChild("Head")

        if not humanoid or humanoid.Health <= 0 or not head then
            continue
        end

        if (head.Position - camera.CFrame.Position).Magnitude > MAX_DISTANCE then
            continue
        end

        local screenPos, visible =
            camera:WorldToViewportPoint(head.Position)

        if not visible then continue end

        local point = Vector2.new(screenPos.X, screenPos.Y)
        local distance = (point - center).Magnitude

        if distance <= FOV_RADIUS
            and canSeeTarget(character, head) then
            table.insert(targets, {
                head = head,
                distance = distance
            })
        end
    end

    table.sort(targets, function(a, b)
        return a.distance < b.distance
    end)

    return targets
end

-- AIM LOOP
RunService.RenderStepped:Connect(function()
    if not aiming then return end

    local targets = getTargets()

    if #targets == 0 then
        currentTarget = nil
        switchRequested = false
        return
    end

    if switchRequested then
        switchRequested = false

        local index = 0
        for i, item in ipairs(targets) do
            if item.head == currentTarget then
                index = i
                break
            end
        end

        index += 1
        if index > #targets then index = 1 end
        currentTarget = targets[index].head

    elseif not currentTarget or not currentTarget.Parent then
        currentTarget = targets[1].head
    else
        local valid = false
        for _, item in ipairs(targets) do
            if item.head == currentTarget then
                valid = true
                break
            end
        end

        if not valid then
            currentTarget = targets[1].head
        end
    end

    if currentTarget and currentTarget.Parent then
        local targetCFrame = CFrame.lookAt(
            camera.CFrame.Position,
            currentTarget.Position
        )

        camera.CFrame = camera.CFrame:Lerp(
            targetCFrame,
            AIM_STRENGTH
        )
    end
end)
