local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- Load Rayfield UI Library
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- Variables
local orbiting = true
local orbitSpeed = 500
local orbitRadius = 3
local orbitConnection
local currentTarget
local humanoidRootPart
local selectedPlayer = nil

local xOffset = 0
local yOffset = 0

local isTeleportedToSky = false
local originalPosition = nil
local touchInterestRemoved = false
local awakeningEnabled = false
local awakeningConnection = nil

-- Anti-Stun variables
local antiStunEnabled = false
local antiStunConnection = nil

-- Dash Length variables (IMPROVED VERSION FROM SECOND SCRIPT)
local DashEnabled = false
local DashConnection = nil
local DashLenghtDistance420 = 5 -- Default value

local FastAttackRange = 5000
local FastAttackEnabled = true
local FastAttackConnection = nil

local Net = ReplicatedStorage:WaitForChild("Modules"):WaitForChild("Net")
local RegisterHit = Net["RE/RegisterHit"]
local RegisterAttack = Net["RE/RegisterAttack"]

-- Create Rayfield Window
local Window = Rayfield:CreateWindow({
    Name = "Silver Hub",
    LoadingTitle = "Silver Hub Loading",
    LoadingSubtitle = "by Silver",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "SkidHub",
        FileName = "SkidHubConfig"
    },
    Discord = {
        Enabled = false
    },
    KeySystem = false
})

-- Optimized attack function
local function AttackMultipleTargets(targets)
    pcall(function()
        if not targets or #targets == 0 then return end
        local allTargets = {}
        for _, targetChar in pairs(targets) do
            local head = targetChar:FindFirstChild("Head")
            local torso = targetChar:FindFirstChild("Torso") or targetChar:FindFirstChild("UpperTorso")
            local hrp = targetChar:FindFirstChild("HumanoidRootPart")
            if head then table.insert(allTargets, {targetChar, head}) end
            if torso and torso ~= head then table.insert(allTargets, {targetChar, torso}) end
            if hrp and hrp ~= head and hrp ~= torso then table.insert(allTargets, {targetChar, hrp}) end
        end
        if #allTargets == 0 then return end
        for i = 1, 3 do
            RegisterAttack:FireServer(0)
            RegisterHit:FireServer(allTargets[1][2], allTargets)
        end
    end)
end

-- Anti-Stun function
local function toggleAntiStun(value)
    antiStunEnabled = value
    if antiStunEnabled then
        antiStunConnection = RunService.Heartbeat:Connect(function()
            local character = player.Character
            if not character then return end
            
            local humanoid = character:FindFirstChild("Humanoid")
            local hrp = character:FindFirstChild("HumanoidRootPart")
            
            if humanoid and hrp then
                if humanoid:GetState() == Enum.HumanoidStateType.Seated then
                    humanoid:ChangeState(Enum.HumanoidStateType.Running)
                end
                
                for _, obj in pairs(hrp:GetChildren()) do
                    if obj:IsA("BodyPosition") or obj:IsA("BodyVelocity") or obj:IsA("BodyGyro") then
                        obj:Destroy()
                    end
                end
                
                if hrp.AssemblyLinearVelocity.Magnitude > 50 then
                    hrp.AssemblyLinearVelocity = Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0)
                end
            end
        end)
    else
        if antiStunConnection then
            antiStunConnection:Disconnect()
            antiStunConnection = nil
        end
    end
end

local function toggleAwakening(value)
    awakeningEnabled = value
    if awakeningEnabled then
        awakeningConnection = task.spawn(function()
            while awakeningEnabled do
                task.wait(0.5)
                pcall(function()
                    local backpack = player:WaitForChild("Backpack", 2)
                    if backpack then
                        local awakening = backpack:FindFirstChild("Awakening")
                        if awakening then
                            local remoteFunc = awakening:FindFirstChild("RemoteFunction")
                            if remoteFunc then remoteFunc:InvokeServer(true) end
                        end
                    end
                end)
            end
        end)
    else
        if awakeningConnection then task.cancel(awakeningConnection) awakeningConnection = nil end
    end
end

-- Fast attack function
local function toggleFastAttack(value)
    FastAttackEnabled = value
    if FastAttackEnabled then
        FastAttackConnection = task.spawn(function()
            while FastAttackEnabled do
                task.wait(0.01)
                local myChar = player.Character
                local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
                if not myHRP then continue end
                local targetsInRange = {}
                local enemiesFolder = workspace:FindFirstChild("Enemies")
                if enemiesFolder then
                    for _, npc in pairs(enemiesFolder:GetChildren()) do
                        local humanoid = npc:FindFirstChild("Humanoid")
                        local hrp = npc:FindFirstChild("HumanoidRootPart")
                        if humanoid and hrp and humanoid.Health > 0 then
                            if (hrp.Position - myHRP.Position).Magnitude <= FastAttackRange then
                                table.insert(targetsInRange, npc)
                            end
                        end
                    end
                end
                for _, p in pairs(Players:GetPlayers()) do
                    if p ~= player and p.Character then
                        local humanoid = p.Character:FindFirstChild("Humanoid")
                        local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                        if humanoid and hrp and humanoid.Health > 0 then
                            if (hrp.Position - myHRP.Position).Magnitude <= FastAttackRange then
                                table.insert(targetsInRange, p.Character)
                            end
                        end
                    end
                end
                if #targetsInRange > 0 then AttackMultipleTargets(targetsInRange) end
            end
        end)
    else
        if FastAttackConnection then task.cancel(FastAttackConnection) FastAttackConnection = nil end
    end
end

local function removeTouchInterest()
    if player.Character then
        for _, part in pairs(player.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false part.CanTouch = false part.CanQuery = false end
        end
    end
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") then pcall(function() obj.CanTouch = false obj.CanQuery = false end) end
    end
end

local function restoreTouchInterest()
    if player.Character then
        for _, part in pairs(player.Character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = part.Name ~= "HumanoidRootPart"
                part.CanTouch = true
                part.CanQuery = true
            end
        end
    end
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") then pcall(function() obj.CanTouch = true obj.CanQuery = true end) end
    end
end

local function toggleTouchInterest(value)
    touchInterestRemoved = not value
    if touchInterestRemoved then
        removeTouchInterest()
    else
        restoreTouchInterest()
    end
end

local function startOrbit()
    if not selectedPlayer then return end
    if not selectedPlayer.Character or not selectedPlayer.Character:FindFirstChild("HumanoidRootPart") then return end
    if not humanoidRootPart or not humanoidRootPart.Parent then return end
    
    if orbitConnection then orbitConnection:Disconnect() end
    
    orbiting = true
    currentTarget = selectedPlayer.Character.HumanoidRootPart
    
    orbitConnection = RunService.RenderStepped:Connect(function()
        if not humanoidRootPart or not humanoidRootPart.Parent then return end
        if not selectedPlayer or not selectedPlayer.Character then return end
        
        currentTarget = selectedPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not currentTarget or not currentTarget.Parent then return end
        
        local angle = tick() * orbitSpeed
        local offset = Vector3.new(math.cos(angle) * orbitRadius + xOffset, yOffset, math.sin(angle) * orbitRadius)
        humanoidRootPart.CFrame = CFrame.lookAt(currentTarget.Position + offset, currentTarget.Position)
        
        local targetHumanoid = currentTarget.Parent:FindFirstChild("Humanoid")
        if targetHumanoid then 
            camera.CameraSubject = targetHumanoid 
        end
    end)
end

local function stopOrbit()
    orbiting = false
    if orbitConnection then 
        orbitConnection:Disconnect() 
        orbitConnection = nil
    end
    camera.CameraSubject = player.Character and player.Character:FindFirstChild("Humanoid")
end

local function toggleOrbit(value)
    if value then
        if selectedPlayer then
            startOrbit()
        end
    else
        stopOrbit()
    end
end

local function toggleTeleportToSky()
    if not humanoidRootPart or not humanoidRootPart.Parent then return end
    if isTeleportedToSky then
        if originalPosition then humanoidRootPart.CFrame = CFrame.new(originalPosition) end
        isTeleportedToSky = false
    else
        originalPosition = humanoidRootPart.Position
        humanoidRootPart.CFrame = CFrame.new(humanoidRootPart.Position.X, 99999999999, humanoidRootPart.Position.Z)
        isTeleportedToSky = true
    end
end

local function teleportToBarcode()
    if not humanoidRootPart or not humanoidRootPart.Parent then return end
    humanoidRootPart.CFrame = CFrame.new(-6504.08, 128.74, -122.46)
end

local function onCharacterAdded(char)
    humanoidRootPart = char:WaitForChild("HumanoidRootPart")
    isTeleportedToSky = false
    originalPosition = nil
    
    if touchInterestRemoved then task.wait(0.1) removeTouchInterest() end
    if antiStunEnabled then toggleAntiStun(false) task.wait(0.1) toggleAntiStun(true) end
    if not orbiting then camera.CameraSubject = char:FindFirstChild("Humanoid") end
end

if player.Character then onCharacterAdded(player.Character) end
player.CharacterAdded:Connect(onCharacterAdded)

-- Create Tabs
local MainTab = Window:CreateTab("Main", 4483362458)
local CombatTab = Window:CreateTab("Combat", 4483362458)
local OrbitTab = Window:CreateTab("Orbit", 4483362458)

-- Main Tab Elements
local TeleportSection = MainTab:CreateSection("Teleportation")

local TeleportSkyButton = MainTab:CreateButton({
    Name = "Teleport to Sky",
    Callback = function()
        toggleTeleportToSky()
    end,
})

local BarcodeButton = MainTab:CreateButton({
    Name = "Teleport to Haunted",
    Callback = function()
        teleportToBarcode()
    end,
})

local MiscSection = MainTab:CreateSection("Miscellaneous")

local TouchToggle = MainTab:CreateToggle({
    Name = "Touch Interest",
    CurrentValue = true,
    Flag = "TouchToggle",
    Callback = function(Value)
        toggleTouchInterest(Value)
    end,
})

local AwakeningToggle = MainTab:CreateToggle({
    Name = "Auto V4",
    CurrentValue = false,
    Flag = "AwakeningToggle",
    Callback = function(Value)
        toggleAwakening(Value)
    end,
})

-- Combat Tab Elements
local CombatSection = CombatTab:CreateSection("Combat Features")

local FastAttackToggle = CombatTab:CreateToggle({
    Name = "Fast Attack",
    CurrentValue = false,
    Flag = "FastAttackToggle",
    Callback = function(Value)
        toggleFastAttack(Value)
    end,
})

local AntiStunToggle = CombatTab:CreateToggle({
    Name = "Anti Stun",
    CurrentValue = false,
    Flag = "AntiStunToggle",
    Callback = function(Value)
        toggleAntiStun(Value)
    end,
})

local DashSection = CombatTab:CreateSection("Dash Settings")

-- IMPROVED DASH LENGTH FROM SECOND SCRIPT
local DashLenght = CombatTab:CreateDropdown({
    Name = "Dash Distance Value",
    Options = {"180", "120", "90", "60", "35", "5"},
    CurrentOption = {"5"},
    MultipleOptions = false,
    Flag = "DashLenght",
    Callback = function(Dashii)
        if type(Dashii) == "table" then Dashii = Dashii[1] end
        DashLenghtDistance420 = tonumber(Dashii) or 5
    end
})

CombatTab:CreateToggle({
    Name = "Dash Length",
    CurrentValue = false,
    Flag = "DashLengthToggle",
    Callback = function(dashsh)
        DashEnabled = dashsh
        if dashsh then
            if DashConnection then task.cancel(DashConnection) end
            
            DashConnection = task.spawn(function()
                while DashEnabled do
                    task.wait(0.1)
                    local character = Players.LocalPlayer.Character
                    if character then
                        local currentValue = character:GetAttribute("DashLength")
                        if currentValue ~= DashLenghtDistance420 then
                            character:SetAttribute("DashLength", DashLenghtDistance420)
                            character:SetAttribute("DashLengthAir", DashLenghtDistance420)
                        end
                    end
                end
            end)
        else
            if DashConnection then 
                local character = Players.LocalPlayer.Character
                if character then
                    character:SetAttribute("DashLength", 1)
                    character:SetAttribute("DashLengthAir", 1)
                end
                task.cancel(DashConnection)
                DashConnection = nil
            end
        end
    end
})

-- Orbit Tab Elements
local OrbitSection = OrbitTab:CreateSection("Orbit Settings")

local XOffsetSlider = OrbitTab:CreateSlider({
    Name = "X Offset",
    Range = {-500, 500},
    Increment = 1,
    CurrentValue = 0,
    Flag = "XOffset",
    Callback = function(Value)
        xOffset = Value
    end,
})

local YOffsetSlider = OrbitTab:CreateSlider({
    Name = "Y Offset",
    Range = {-500, 500},
    Increment = 1,
    CurrentValue = 0,
    Flag = "YOffset",
    Callback = function(Value)
        yOffset = Value
    end,
})

local PlayerSection = OrbitTab:CreateSection("Player Selection")

local OrbitToggle = OrbitTab:CreateToggle({
    Name = "Enable Orbit",
    CurrentValue = false,
    Flag = "OrbitToggle",
    Callback = function(Value)
        toggleOrbit(Value)
    end,
})

-- Create player dropdown
local playerNames = {}
local playerDropdown

local function updatePlayerList()
    playerNames = {}
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player then
            table.insert(playerNames, p.Name)
        end
    end
    
    if playerDropdown then
        playerDropdown:Refresh(playerNames)
    end
end

playerDropdown = OrbitTab:CreateDropdown({
    Name = "Select Player to Orbit",
    Options = playerNames,
    CurrentOption = "",
    Flag = "PlayerDropdown",
    Callback = function(Option)
        local targetPlayer = Players:FindFirstChild(Option)
        if targetPlayer then
            selectedPlayer = targetPlayer
            if orbiting then
                startOrbit()
            end
        end
    end,
})

-- Update player list on player added/removed
Players.PlayerAdded:Connect(updatePlayerList)
Players.PlayerRemoving:Connect(updatePlayerList)
updatePlayerList()

-- Keyboard shortcut
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.N then toggleTeleportToSky() end
end)

-- Notification
Rayfield:Notify({
    Title = "Silver Hub",
    Content = "Script loaded successfully!",
    Duration = 3,
    Image = 4483362458,
})
