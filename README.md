-- Invisibilidade com clone rápido e corpo real ultra distante

local Player = game.Players.LocalPlayer
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera

local RealCharacter = Player.Character or Player.CharacterAdded:Wait()
local Humanoid = RealCharacter:FindFirstChildOfClass("Humanoid")

local IsInvisible = false
local FakeCharacter
local bodyVelocity
local renderConnection

local function cleanup()
    if FakeCharacter and FakeCharacter.Parent then
        FakeCharacter:Destroy()
    end
    if renderConnection then
        renderConnection:Disconnect()
        renderConnection = nil
    end
    workspace.CurrentCamera.CameraSubject = RealCharacter:FindFirstChildOfClass("Humanoid") or Humanoid
    IsInvisible = false
end

local function onCharacterAdded(char)
    RealCharacter = char
    Humanoid = char:WaitForChild("Humanoid")
    cleanup()
end

Player.CharacterAdded:Connect(onCharacterAdded)

local function CreateClone()
    cleanup()

    RealCharacter.Archivable = true
    FakeCharacter = RealCharacter:Clone()

    for _, v in pairs(FakeCharacter:GetDescendants()) do
        if v:IsA("BodyVelocity") or v:IsA("BodyGyro") or v:IsA("BodyPosition") then
            v:Destroy()
        end
        if v:IsA("BasePart") then
            v.Transparency = 0.85
        end
    end

    bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.MaxForce = Vector3.new(500000, 500000, 500000)
    bodyVelocity.Velocity = Vector3.new(0,0,0)
    bodyVelocity.Parent = FakeCharacter.HumanoidRootPart

    FakeCharacter.Parent = workspace
    FakeCharacter.HumanoidRootPart.CFrame = RealCharacter.HumanoidRootPart.CFrame
    FakeCharacter.HumanoidRootPart.Anchored = false
    FakeCharacter.Humanoid.WalkSpeed = RealCharacter.Humanoid.WalkSpeed * 5 -- clone rápido

    local RealAnimator = RealCharacter:FindFirstChildOfClass("Animator")
    if RealAnimator then
        local CloneAnimator = RealAnimator:Clone()
        CloneAnimator.Parent = FakeCharacter
    end

    for _, v in pairs(RealCharacter:GetChildren()) do
        if v:IsA("LocalScript") then
            local clone = v:Clone()
            clone.Disabled = true
            clone.Parent = FakeCharacter
        end
    end

    workspace.CurrentCamera.CameraSubject = FakeCharacter.Humanoid

    -- Corpo real vai tão longe que não renderiza mais
    local distantCFrame = CFrame.new(1e30, 1e30, 1e30)
    IsInvisible = true

    spawn(function()
        while IsInvisible do
            RealCharacter.HumanoidRootPart.CFrame = distantCFrame
            task.wait(0.1)
        end
    end)

    renderConnection = RunService.RenderStepped:Connect(function()
        if not IsInvisible or not FakeCharacter or not FakeCharacter.Parent then return end
        local moveDirection2 = Humanoid.MoveDirection

        if moveDirection2.Magnitude > 0 then
            bodyVelocity.Parent = nil
            local moveDirection = Camera.CFrame.LookVector
            local isMovingBackward = moveDirection:Dot(moveDirection2) < 0
            local adjustedY = isMovingBackward and (moveDirection.Y * -1.4) or (moveDirection.Y * 1.8)
            local adjustedV = isMovingBackward and (2.0) or (2.5)

            FakeCharacter.HumanoidRootPart.Velocity = Vector3.new(moveDirection2.X, adjustedY + 0.35, moveDirection2.Z).Unit * (FakeCharacter.Humanoid.WalkSpeed * adjustedV)
        else
            bodyVelocity.Parent = FakeCharacter.HumanoidRootPart
        end
    end)
end

local function TeleportAndRemoveClone()
    if IsInvisible and FakeCharacter and FakeCharacter.Parent then
        RealCharacter.HumanoidRootPart.Velocity = Vector3.new(0, 0, 0)

        for _, v in pairs(FakeCharacter:GetDescendants()) do
            if v:IsA("BasePart") then
                v.Transparency = 1
            end
        end

        IsInvisible = false

        for _ = 1, 5 do
            RealCharacter.HumanoidRootPart.CFrame = FakeCharacter.HumanoidRootPart.CFrame
            task.wait()
        end

        cleanup()
    end
end

-- Botão pequeno flutuante arrastável

local gui = Instance.new("ScreenGui", Player:WaitForChild("PlayerGui"))
gui.ResetOnSpawn = false
gui.Name = "InvisButtonGui"

local button = Instance.new("TextButton")
button.Size = UDim2.new(0, 80, 0, 30)
button.Position = UDim2.new(0.05, 0, 0.7, 0)
button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
button.TextColor3 = Color3.new(1, 1, 1)
button.Font = Enum.Font.GothamSemibold
button.TextSize = 11
button.Text = "Invisível"
button.Parent = gui

local corner = Instance.new("UICorner", button)
corner.CornerRadius = UDim.new(1, 0)

-- Arrastável (toque e mouse)
local dragging, dragInput, dragStart, startPos

button.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		startPos = button.Position

		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

button.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		dragInput = input
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if input == dragInput and dragging then
		local delta = input.Position - dragStart
		button.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
	end
end)

-- Clique no botão
button.MouseButton1Click:Connect(function()
    if not IsInvisible then
        CreateClone()
        button.Text = "Visível"
        button.BackgroundColor3 = Color3.fromRGB(255, 80, 80)
    else
        TeleportAndRemoveClone()
        button.Text = "Invisível"
        button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    end
end)
