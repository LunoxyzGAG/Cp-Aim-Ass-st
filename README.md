
-- DHEN AIM ASSIST + GOOD GRAPHICS + TOGGLE
-- LocalScript
-- StarterPlayer > StarterPlayerScripts
-- No Menu / No GUI
-- T = Aim ON/OFF
-- Y = Switch Target

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- AIM SETTINGS
local FOV_RADIUS = 100
local AIM_STRENGTH = 0.14
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

local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
if atmosphere then
	atmosphere.Density = 0.02
	atmosphere.Haze = 0.05
	atmosphere.Glare = 0
end

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

-- INPUT: NO GUI
UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end

	if input.KeyCode == Enum.KeyCode.T then
		aiming = not aiming
		if not aiming then
			currentTarget = nil
		end
		print("Aim Assist:", aiming and "ON" or "OFF")

	elseif input.KeyCode == Enum.KeyCode.Y then
		switchRequested = true
	end
end)

-- WALL CHECK
local function canSeeTarget(character, part)
	local origin = camera.CFrame.Position
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = {
		player.Character,
		camera
	}

	local result = workspace:Raycast(
		origin,
		part.Position - origin,
		params
	)

	return result == nil
		or result.Instance:IsDescendantOf(character)
end

-- VALIDATE TARGET
local function isValidTarget(head)
	if not head or not head.Parent then
		return false
	end

	local character = head.Parent
	local humanoid = character:FindFirstChildOfClass("Humanoid")
	if not humanoid or humanoid.Health <= 0 then
		return false
	end

	local distance = (head.Position - camera.CFrame.Position).Magnitude
	if distance > MAX_DISTANCE then
		return false
	end

	local screenPos, visible = camera:WorldToViewportPoint(head.Position)
	if not visible then
		return false
	end

	local center = camera.ViewportSize / 2
	local point = Vector2.new(screenPos.X, screenPos.Y)
	if (point - center).Magnitude > FOV_RADIUS then
		return false
	end

	return canSeeTarget(character, head)
end

-- FIND TARGETS ORDERED BY SCREEN DISTANCE
local function getTargets()
	local targets = {}
	local center = camera.ViewportSize / 2

	for _, other in ipairs(Players:GetPlayers()) do
		if other == player then continue end

		local character = other.Character
		local head = character and character:FindFirstChild("Head")

		if head and isValidTarget(head) then
			local pos = camera:WorldToViewportPoint(head.Position)
			local point = Vector2.new(pos.X, pos.Y)

			table.insert(targets, {
				head = head,
				distance = (point - center).Magnitude
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

	local selected = nil

	if switchRequested then
		switchRequested = false

		local currentIndex = 0
		for i, item in ipairs(targets) do
			if item.head == currentTarget then
				currentIndex = i
				break
			end
		end

		local nextIndex = currentIndex + 1
		if nextIndex > #targets then
			nextIndex = 1
		end
		selected = targets[nextIndex].head
	else
		if isValidTarget(currentTarget) then
			selected = currentTarget
		else
			selected = targets[1].head
		end
	end

	currentTarget = selected

	if currentTarget and isValidTarget(currentTarget) then
		local targetCFrame = CFrame.lookAt(
			camera.CFrame.Position,
			currentTarget.Position
		)

		camera.CFrame = camera.CFrame:Lerp(
			targetCFrame,
			math.clamp(AIM_STRENGTH, 0, 1)
		)
	end
end)
