-- =================================================================
-- AK HUB - ABA ADMIN FUNCIONAL PARA Noob_prime1000q
-- Executores: Delta / Hydrogen / Codex / Fluxus
-- =================================================================

local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")
local GuiService = game:GetService("GuiService")
local StarterGui = game:GetService("StarterGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- Nome do Dono Autorizado
local OWNER_NAME = "Noob_prime1000q"
local isOwner = (LocalPlayer.Name == OWNER_NAME)

-- Remove UI antiga caso já exista
if CoreGui:FindFirstChild("AKHubUI") then
    CoreGui.AKHubUI:Destroy()
end

-- Cor padrão unificada para os botões
local DEFAULT_BTN_COLOR = Color3.fromRGB(60, 60, 80)
local ACTIVE_BTN_COLOR = Color3.fromRGB(50, 150, 80)
local WORKING_BTN_COLOR = Color3.fromRGB(200, 140, 40)

-- Variáveis Globais de Controle
local isMainDraggable = true
local activeAccentColor = Color3.fromRGB(100, 100, 255)
local selectedTrollPlayer = nil
local selectedAdminPlayer = nil
local globalActionLock = false

-- Variáveis do Fling / Annoy / Bloco
local activeBlockSystem = false
local isAnnoyMode = false
local targetBlock = nil
local blockConnection = nil
local origCollide = nil
local MAX_RANGE = 300
local targetPartName = "HumanoidRootPart"
local useDoorMode = false
local originalPosBeforeDoorFling = nil

-- Variáveis do Kill / Bring / All Bring
local killRunning = false
local bringRunning = false
local allBringRunning = false
local DELETE_IMG_ID = "6893077263"

-- Variável do View (Spectate)
local viewing = false
local viewConn = nil

-- Variáveis de Proteções (Estados)
local antiFlingActive = false
local antiKickActive = false
local antiVoidActive = false
local antiSitActive = false

local antiFlingConn = nil
local antiVoidConn = nil
local antiSitConn = nil

-- Variáveis da Aba Carro
local carSpeedActive = false
local carTurboActive = false
local customCarSpeed = 3
local customTurboSpeed = 6
local carConnection = nil

-- =================================================================
-- SISTEMA DA ABA CARRO (MODIFICAÇÃO NATIVA DO BROOKHAVEN)
-- =================================================================
local function ApplyBrookhavenVehicleMods()
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    
    if hum and hum.SeatPart and hum.SeatPart:IsA("VehicleSeat") then
        local seat = hum.SeatPart
        local carModel = seat:FindFirstAncestorOfClass("Model")
        
        local finalMult = carTurboActive and customTurboSpeed or customCarSpeed
        local targetMaxSpeed = math.clamp(70 * finalMult, 50, 500)

        pcall(function()
            seat.MaxSpeed = targetMaxSpeed
            seat.Torque = 100 * finalMult
        end)

        if carModel then
            for _, child in ipairs(carModel:GetDescendants()) do
                if child:IsA("NumberValue") or child:IsA("IntValue") then
                    local nameLower = string.lower(child.Name)
                    if string.find(nameLower, "speed") or string.find(nameLower, "maxspeed") or string.find(nameLower, "velocity") then
                        pcall(function() child.Value = targetMaxSpeed end)
                    end
                end
            end
        end

        if seat.ThrottleFloat ~= 0 then
            local currentVel = seat.AssemblyLinearVelocity
            local flatDirection = Vector3.new(seat.CFrame.LookVector.X, 0, seat.CFrame.LookVector.Z).Unit
            
            if math.abs(currentVel.Y) < 6 then
                seat.AssemblyLinearVelocity = Vector3.new(
                    flatDirection.X * (targetMaxSpeed * seat.ThrottleFloat),
                    currentVel.Y,
                    flatDirection.Z * (targetMaxSpeed * seat.ThrottleFloat)
                )
            end
        end
    end
end

local function ToggleCarSpeed(state)
    carSpeedActive = state
    if carSpeedActive then
        carConnection = RunService.RenderStepped:Connect(function()
            if carSpeedActive then
                ApplyBrookhavenVehicleMods()
            end
        end)
    else
        if carConnection then
            carConnection:Disconnect()
            carConnection = nil
        end
        pcall(function()
            local char = LocalPlayer.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum and hum.SeatPart and hum.SeatPart:IsA("VehicleSeat") then
                hum.SeatPart.MaxSpeed = 70
            end
        end)
    end
end

local function ToggleCarTurbo(state)
    carTurboActive = state
end

-- =================================================================
-- SISTEMA DE PROTEÇÕES
-- =================================================================
local function ToggleAntiFling(state)
    antiFlingActive = state
    if antiFlingActive then
        antiFlingConn = RunService.Stepped:Connect(function()
            local char = LocalPlayer.Character
            if char then
                for _, part in ipairs(char:GetDescendants()) do
                    if part:IsA("BasePart") then
                        part.CustomPhysicalProperties = PhysicalProperties.new(100, 0, 0, 100, 100)
                    end
                end
            end
        end)
    else
        if antiFlingConn then antiFlingConn:Disconnect() antiFlingConn = nil end
    end
end

local function ToggleAntiKick(state)
    antiKickActive = state
    if antiKickActive then
        local mt = getrawmetatable(game)
        if mt then
            setreadonly(mt, false)
            local oldNamecall = mt.__namecall
            mt.__namecall = newcclosure(function(self, ...)
                local method = getnamecallmethod()
                if antiKickActive and method == "Kick" and self == LocalPlayer then
                    return
                end
                return oldNamecall(self, ...)
            end)
            setreadonly(mt, true)
        end
    end
end

local function ToggleAntiVoid(state)
    antiVoidActive = state
    if antiVoidActive then
        antiVoidConn = RunService.Heartbeat:Connect(function()
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp and hrp.Position.Y < -50 and not killRunning and not bringRunning and not allBringRunning then
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.CFrame = CFrame.new(hrp.Position.X, 50, hrp.Position.Z)
            end
        end)
    else
        if antiVoidConn then antiVoidConn:Disconnect() antiVoidConn = nil end
    end
end

local function ToggleAntiSit(state)
    antiSitActive = state
    if antiSitActive then
        antiSitConn = RunService.Heartbeat:Connect(function()
            local char = LocalPlayer.Character
            local humanoid = char and char:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid.Sit = false
            end
        end)
    else
        if antiSitConn then antiSitConn:Disconnect() antiSitConn = nil end
    end
end

-- =================================================================
-- FUNÇÕES DE CLIQUE GUI E IMAGENS
-- =================================================================
local IS_TOUCH = UserInputService.TouchEnabled
local function clickAt(x, y)
    pcall(function()
        if IS_TOUCH then
            VirtualInputManager:SendTouchEvent(1, 0, x, y)
            task.wait(0.05)
            VirtualInputManager:SendTouchEvent(1, 2, x, y)
        else
            VirtualInputManager:SendMouseButtonEvent(x, y, 0, true, game, 0)
            task.wait(0.05)
            VirtualInputManager:SendMouseButtonEvent(x, y, 0, false, game, 0)
        end
    end)
end

local function isShown(g)
    local o = g
    while o and o ~= game do
        if o:IsA("GuiObject") and not o.Visible then return false end
        if o:IsA("ScreenGui") and not o.Enabled then return false end
        o = o.Parent
    end
    return true
end

local function findImage(id)
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if not pg then return nil end
    local fallback
    for _, g in ipairs(pg:GetDescendants()) do
        if (g:IsA("ImageButton") or g:IsA("ImageLabel")) and string.find(g.Image, id, 1, true) then
            if isShown(g) then return g end
            fallback = fallback or g
        end
    end
    return fallback
end

local function clickGui(g)
    local btn = g
    while btn and not btn:IsA("GuiButton") do btn = btn.Parent end
    if btn and firesignal then
        local ok = pcall(function()
            firesignal(btn.MouseButton1Click)
            firesignal(btn.Activated)
        end)
        if ok then return end
    end
    local pos = g.AbsolutePosition + g.AbsoluteSize / 2
    local inset = GuiService:GetGuiInset()
    clickAt(pos.X + inset.X, pos.Y + inset.Y)
end

local function clickImage(id)
    for attempt = 1, 3 do
        local g = findImage(id)
        if g then
            clickGui(g)
            return true
        end
        task.wait(0.15)
    end
    return false
end

-- =================================================================
-- CÓDIGO DA FUNÇÃO ;KILL E ;BRING
-- =================================================================
local function ExecuteCouchKill(targetPlayer)
    if killRunning or globalActionLock then return false end
    killRunning = true
    globalActionLock = true

    local previousAntiSitState = antiSitActive
    if not antiSitActive then ToggleAntiSit(true) end

    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")

    if not char or not hrp or not hum then
        ToggleAntiSit(previousAntiSitState)
        killRunning = false
        globalActionLock = false
        return false
    end

    local originalCFrame = hrp.CFrame
    hrp.CFrame = CFrame.new(-82.6, 22.1, -130.0)
    task.wait(0.2)

    hum.Jump = true
    task.wait(0.25)

    local toolName = "Couch"
    local couchTool = char:FindFirstChild(toolName) or (LocalPlayer:FindFirstChild("Backpack") and LocalPlayer.Backpack:FindFirstChild(toolName))
    if couchTool then hum:EquipTool(couchTool) end
    task.wait(0.2)

    local targetChar = targetPlayer.Character
    local targetHrp = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
    local targetHum = targetChar and targetChar:FindFirstChildOfClass("Humanoid")

    if not targetChar or not targetHrp or not targetHum then
        ToggleAntiSit(previousAntiSitState)
        killRunning = false
        globalActionLock = false
        return false
    end

    local loopTimeout = 0
    local hasSat = false

    while killRunning and loopTimeout < 150 do
        if not targetChar or not targetHrp or not targetHum or targetHum.Health <= 0 then break end
        hum.Sit = false
        hrp.CFrame = targetHrp.CFrame * CFrame.new(0, -3.6, 0)
        if targetHum.Sit then hasSat = true break end
        task.wait(0.03)
        loopTimeout = loopTimeout + 1
    end

    if hasSat then
        task.wait(0.05)
        local savedHealth = hum.Health
        pcall(function()
            hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
            hum.Health = 0
        end)
        hrp.CFrame = CFrame.new(hrp.Position.X, -350, hrp.Position.Z)
        task.wait(1.0)
        clickImage(DELETE_IMG_ID)
        task.wait(0.25)
        pcall(function()
            hum.Health = savedHealth > 0 and savedHealth or 100
            hum:SetStateEnabled(Enum.HumanoidStateType.Dead, true)
        end)
    end

    if char and hrp then hrp.CFrame = originalCFrame end
    ToggleAntiSit(previousAntiSitState)
    killRunning = false
    globalActionLock = false
    return hasSat
end

local function ExecuteCouchBring(targetPlayer)
    if bringRunning or globalActionLock then return false end
    bringRunning = true
    globalActionLock = true

    local previousAntiSitState = antiSitActive
    if not antiSitActive then ToggleAntiSit(true) end

    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")

    if not char or not hrp or not hum then
        ToggleAntiSit(previousAntiSitState)
        bringRunning = false
        globalActionLock = false
        return false
    end

    local originalCFrame = hrp.CFrame
    hrp.CFrame = CFrame.new(-82.6, 22.1, -130.0)
    task.wait(0.2)

    hum.Jump = true
    task.wait(0.25)

    local toolName = "Couch"
    local couchTool = char:FindFirstChild(toolName) or (LocalPlayer:FindFirstChild("Backpack") and LocalPlayer.Backpack:FindFirstChild(toolName))
    if couchTool then hum:EquipTool(couchTool) end
    task.wait(0.2)

    local targetChar = targetPlayer.Character
    local targetHrp = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
    local targetHum = targetChar and targetChar:FindFirstChildOfClass("Humanoid")

    if not targetChar or not targetHrp or not targetHum then
        ToggleAntiSit(previousAntiSitState)
        bringRunning = false
        globalActionLock = false
        return false
    end

    local loopTimeout = 0
    local hasSat = false

    while bringRunning and loopTimeout < 150 do
        if not targetChar or not targetHrp or not targetHum or targetHum.Health <= 0 then break end
        hum.Sit = false
        hrp.CFrame = targetHrp.CFrame * CFrame.new(0, -3.6, 0)
        if targetHum.Sit then hasSat = true break end
        task.wait(0.03)
        loopTimeout = loopTimeout + 1
    end

    if hasSat then
        task.wait(0.05)
        hrp.CFrame = originalCFrame
        task.wait(0.4)
        clickImage(DELETE_IMG_ID)
        task.wait(0.25)
    end

    if char and hrp then hrp.CFrame = originalCFrame end
    ToggleAntiSit(previousAntiSitState)
    bringRunning = false
    globalActionLock = false
    return hasSat
end

-- =================================================================
-- SEQUÊNCIA DA BOLA (EXCLUSIVA DO FLING / ANNOY)
-- =================================================================
local IMG_1 = "5084437088"
local IMG_2 = "4598172149"
local TOOL_NAME = "SoccerBall"

local function notify(text, title)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title or "AK Hub",
            Text = text,
            Duration = 4
        })
    end)
end

local function scrollIntoView(g)
    local o = g.Parent
    while o do
        if o:IsA("ScrollingFrame") then
            pcall(function()
                local relY = g.AbsolutePosition.Y - o.AbsolutePosition.Y + o.CanvasPosition.Y
                local relX = g.AbsolutePosition.X - o.AbsolutePosition.X + o.CanvasPosition.X
                local maxY = math.max(0, o.AbsoluteCanvasSize.Y - o.AbsoluteSize.Y)
                local maxX = math.max(0, o.AbsoluteCanvasSize.X - o.AbsoluteSize.X)
                o.CanvasPosition = Vector2.new(
                    math.clamp(relX - o.AbsoluteSize.X / 2 + g.AbsoluteSize.X / 2, 0, maxX),
                    math.clamp(relY - o.AbsoluteSize.Y / 2 + g.AbsoluteSize.Y / 2, 0, maxY))
            end)
            task.wait(0.1)
        end
        o = o.Parent
    end
end

local function sweepMenus(id, alive)
    alive = alive or function() return true end
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if not pg then return nil end
    for _, sf in ipairs(pg:GetDescendants()) do
        if sf:IsA("ScrollingFrame") and isShown(sf) then
            local maxY = math.max(0, sf.AbsoluteCanvasSize.Y - sf.AbsoluteSize.Y)
            local step = math.max(40, sf.AbsoluteSize.Y * 0.7)
            local y = 0
            while alive() do
                pcall(function() sf.CanvasPosition = Vector2.new(sf.CanvasPosition.X, math.min(y, maxY)) end)
                task.wait(0.15)
                local g = findImage(id)
                if g then return g end
                if y >= maxY then break end
                y += step
            end
        end
    end
    return nil
end

local function clickImageWithScroll(id, alive)
    alive = alive or function() return true end
    for attempt = 1, 4 do
        if not alive() then return false end
        local g = findImage(id) or sweepMenus(id, alive)
        if g then
            scrollIntoView(g)
            task.wait(0.15)
            clickGui(g)
            return true
        end
        task.wait(0.4)
    end
    return false
end

local function hasBallInInventory()
    local char = LocalPlayer.Character
    local bag = LocalPlayer:FindFirstChild("Backpack")
    if char and char:FindFirstChild(TOOL_NAME) then return true end
    if bag and bag:FindFirstChild(TOOL_NAME) then return true end
    return false
end

local function equipBall()
    local t = 0
    while t < 5 do
        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            local tool = char:FindFirstChild(TOOL_NAME)
            if tool and tool:IsA("Tool") then return tool end
            local bag = LocalPlayer:FindFirstChild("Backpack")
            local inBag = bag and bag:FindFirstChild(TOOL_NAME)
            if inBag then pcall(function() hum:EquipTool(inBag) end) end
        end
        task.wait(0.25)
        t += 0.25
    end
    return nil
end

local function dropBall()
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    pcall(function() hum:UnequipTools() end)

    local tool = char:FindFirstChild(TOOL_NAME)
    if tool then
        local bag = LocalPlayer:FindFirstChild("Backpack")
        if bag then tool.Parent = bag else tool:Destroy() end
    end
end

local function RunBallSequence()
    if hasBallInInventory() then
        local tool = equipBall()
        if tool then
            task.wait(0.2)
            local vs = Camera.ViewportSize
            clickAt(vs.X / 2, vs.Y / 2)
            task.wait(0.2)
            pcall(function() tool:Activate() end)
            task.wait(0.3)
            dropBall()
            task.wait(0.2)
            return true
        end
    end

    if not clickImageWithScroll(IMG_1) then notify("Imagem 1 não encontrada") end
    task.wait(0.4)
    if not clickImageWithScroll(IMG_2) then notify("Imagem 2 não encontrada") end
    task.wait(0.4)

    local tool = equipBall()
    if not tool then
        notify(TOOL_NAME .. " não encontrado")
        return false
    end
    task.wait(0.3)

    local vs = Camera.ViewportSize
    clickAt(vs.X / 2, vs.Y / 2)
    task.wait(0.3)
    pcall(function() tool:Activate() end)

    task.wait(0.5)
    dropBall()
    task.wait(0.3)
    return true
end

-- =================================================================
-- FUNÇÃO DO VIEW (SPECTATE)
-- =================================================================
local function stopView()
    viewing = false
    if viewConn then viewConn:Disconnect() viewConn = nil end
    Camera.CameraSubject = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
end

local function startView(targetPlayer)
    if not targetPlayer or targetPlayer == LocalPlayer then return end
    viewing = true

    viewConn = RunService.RenderStepped:Connect(function()
        if not viewing or not selectedTrollPlayer or not selectedTrollPlayer.Character then
            stopView()
            return
        end
        local humanoid = selectedTrollPlayer.Character:FindFirstChildOfClass("Humanoid")
        if humanoid then Camera.CameraSubject = humanoid end
    end)
end

-- =================================================================
-- FUNÇÃO: ENCONTRAR BLOCOS / PORTAS E CONTROLE (FLING / ANNOY)
-- =================================================================
local function findNearestBlock()
    local myChar = LocalPlayer.Character
    local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myHRP then return nil end

    local closestPart = nil
    local shortestDistance = MAX_RANGE

    local descendants = workspace:GetDescendants()
    for i = 1, #descendants do
        local p = descendants[i]
        if p:IsA("BasePart") then
            local isValid = true
            if useDoorMode then
                local nameLower = string.lower(p.Name)
                local isDoorName = string.find(nameLower, "door") ~= nil
                local isPhysical = not p.Anchored
                isValid = isDoorName and isPhysical
            else
                local owner = p:FindFirstAncestorOfClass("Model")
                local isPlayerOrNPC = owner and (Players:GetPlayerFromCharacter(owner) or owner:FindFirstChildOfClass("Humanoid"))
                isValid = not p.Anchored and p.Size.Magnitude < 35 and not isPlayerOrNPC
            end

            if isValid then
                local dist = (p.Position - myHRP.Position).Magnitude
                if dist < shortestDistance then
                    shortestDistance = dist
                    closestPart = p
                end
            end
        end
    end

    return closestPart
end

local function startBlockSystem(partName, isDoor, isAnnoy)
    targetPartName = partName or "HumanoidRootPart"
    useDoorMode = isDoor or false
    isAnnoyMode = isAnnoy or false
    activeBlockSystem = true

    pcall(function() sethiddenproperty(LocalPlayer, "SimulationRadius", math.huge) end)
    pcall(function() LocalPlayer.MaximumSimulationRadius = math.huge end)

    targetBlock = findNearestBlock()
    if not targetBlock then
        activeBlockSystem = false
        return false
    end

    local myChar = LocalPlayer.Character
    local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if useDoorMode and myHRP then
        originalPosBeforeDoorFling = myHRP.CFrame
        myHRP.CFrame = targetBlock.CFrame + Vector3.new(0, 3, 0)
        task.wait(0.15)
    end

    origCollide = targetBlock.CanCollide
    pcall(function() targetBlock.CanCollide = true end)

    blockConnection = RunService.Heartbeat:Connect(function()
        if not activeBlockSystem or not targetBlock or not targetBlock.Parent then
            targetBlock = findNearestBlock()
            if not targetBlock then return end
        end

        local targetChar = (selectedTrollPlayer and selectedTrollPlayer.Character) or LocalPlayer.Character
        local targetPart = targetChar and targetChar:FindFirstChild("HumanoidRootPart")

        if targetPart then
            pcall(function()
                local velocity = targetPart.AssemblyLinearVelocity
                local speed = velocity.Magnitude
                local targetPos = targetPart.Position

                if isAnnoyMode then
                    targetBlock.CFrame = CFrame.new(targetPos)
                    targetBlock.CanCollide = true
                    targetBlock.Massless = true

                    targetBlock.AssemblyLinearVelocity = Vector3.new(800, 800, 800)
                    targetBlock.AssemblyAngularVelocity = Vector3.new(0, 50000, 50000)
                else
                    if speed > 2 then
                        targetPos = targetPos + (velocity.Unit * 3)
                    end

                    targetBlock.CFrame = CFrame.new(targetPos)
                    targetBlock.CanCollide = true
                    targetBlock.Massless = false

                    targetBlock.AssemblyLinearVelocity = Vector3.new(50000, 50000, 50000)
                    targetBlock.AssemblyAngularVelocity = Vector3.new(5000, 5000, 5000)
                end
            end)
        end
    end)

    return true
end

local function stopBlockSystem()
    activeBlockSystem = false
    isAnnoyMode = false
    if blockConnection then
        blockConnection:Disconnect()
        blockConnection = nil
    end

    if targetBlock then
        pcall(function()
            targetBlock.CanCollide = origCollide or true
            targetBlock.AssemblyLinearVelocity = Vector3.zero
            targetBlock.AssemblyAngularVelocity = Vector3.zero
        end)
        targetBlock = nil
    end

    if useDoorMode and originalPosBeforeDoorFling then
        local myChar = LocalPlayer.Character
        local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
        if myHRP then
            myHRP.CFrame = originalPosBeforeDoorFling
        end
        originalPosBeforeDoorFling = nil
    end

    useDoorMode = false
end

-- =================================================================
-- 1. CRIANDO A INTERFACE (GUI)
-- =================================================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AKHubUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = CoreGui

-- Botão Flutuante "AK"
local ToggleButton = Instance.new("TextButton")
ToggleButton.Name = "ToggleButton"
ToggleButton.Size = UDim2.new(0, 50, 0, 50)
ToggleButton.Position = UDim2.new(0.05, 0, 0.2, 0)
ToggleButton.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
ToggleButton.Text = "AK"
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.TextSize = 15
ToggleButton.Font = Enum.Font.GothamBold
ToggleButton.Parent = ScreenGui

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(0, 12)
ToggleCorner.Parent = ToggleButton

local ToggleStroke = Instance.new("UIStroke")
ToggleStroke.Color = activeAccentColor
ToggleStroke.Thickness = 2
ToggleStroke.Parent = ToggleButton

local draggingToggle, dragInputToggle, dragStartToggle, startPosToggle
ToggleButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingToggle = true
        dragStartToggle = input.Position
        startPosToggle = ToggleButton.Position
    end
end)

ToggleButton.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInputToggle = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInputToggle and draggingToggle then
        local delta = input.Position - dragStartToggle
        ToggleButton.Position = UDim2.new(startPosToggle.X.Scale, startPosToggle.X.Offset + delta.X, startPosToggle.Y.Scale, startPosToggle.Y.Offset + delta.Y)
    end
end)

-- Janela Principal
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 0, 0, 0)
MainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
MainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
MainFrame.ClipsDescendants = false
MainFrame.Visible = false
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(60, 60, 80)
MainStroke.Thickness = 1.5
MainStroke.Parent = MainFrame

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Name = "TitleLabel"
TitleLabel.Size = UDim2.new(1, 0, 0, 40)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "AK HUB"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.TextSize = 16
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.Active = true
TitleLabel.Parent = MainFrame

local isOpen = false
local originalSize = UDim2.new(0, 450, 0, 300)
local closedSize = UDim2.new(0, 0, 0, 0)

local function toggleMenu()
    isOpen = not isOpen
    if isOpen then
        MainFrame.Visible = true
        TweenService:Create(MainFrame, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = originalSize}):Play()
    else
        local closeTween = TweenService:Create(MainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Size = closedSize})
        closeTween:Play()
        closeTween.Completed:Connect(function()
            if not isOpen then MainFrame.Visible = false end
        end)
    end
end

-- Botão de Fechar (✖) na Janela Principal
local CloseButton = Instance.new("TextButton")
CloseButton.Name = "CloseButton"
CloseButton.Size = UDim2.new(0, 30, 0, 30)
CloseButton.Position = UDim2.new(1, -35, 0, 5)
CloseButton.BackgroundColor3 = Color3.fromRGB(150, 40, 40)
CloseButton.Text = "✖"
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.TextSize = 14
CloseButton.Font = Enum.Font.GothamBold
CloseButton.Parent = MainFrame

Instance.new("UICorner", CloseButton).CornerRadius = UDim.new(0, 6)

CloseButton.MouseButton1Click:Connect(function()
    toggleMenu()
end)

ToggleButton.MouseButton1Click:Connect(toggleMenu)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed and input.KeyCode == Enum.KeyCode.K then toggleMenu() end
end)

-- Barra Lateral de Abas
local TabSidebar = Instance.new("Frame")
TabSidebar.Name = "TabSidebar"
TabSidebar.Size = UDim2.new(0, 110, 1, -40)
TabSidebar.Position = UDim2.new(0, 0, 0, 40)
TabSidebar.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
TabSidebar.BorderSizePixel = 0
TabSidebar.Parent = MainFrame

local SidebarLayout = Instance.new("UIListLayout")
SidebarLayout.SortOrder = Enum.SortOrder.LayoutOrder
SidebarLayout.Padding = UDim.new(0, 5)
SidebarLayout.Parent = TabSidebar

local SidebarPadding = Instance.new("UIPadding")
SidebarPadding.PaddingTop = UDim.new(0, 10)
SidebarPadding.PaddingLeft = UDim.new(0, 5)
SidebarPadding.PaddingRight = UDim.new(0, 5)
SidebarPadding.Parent = TabSidebar

local ContainerPages = Instance.new("Frame")
ContainerPages.Name = "ContainerPages"
ContainerPages.Size = UDim2.new(1, -110, 1, -40)
ContainerPages.Position = UDim2.new(0, 110, 0, 40)
ContainerPages.BackgroundTransparency = 1
ContainerPages.Parent = MainFrame

local Tabs = {}
local function CreateTab(tabName)
    local TabButton = Instance.new("TextButton")
    TabButton.Name = tabName .. "Button"
    TabButton.Size = UDim2.new(1, 0, 0, 30)
    TabButton.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    TabButton.Text = tabName
    TabButton.TextColor3 = Color3.fromRGB(150, 150, 170)
    TabButton.TextSize = 11
    TabButton.Font = Enum.Font.GothamBold
    TabButton.Parent = TabSidebar

    local TabButtonCorner = Instance.new("UICorner")
    TabButtonCorner.CornerRadius = UDim.new(0, 6)
    TabButtonCorner.Parent = TabButton

    local TabPage = Instance.new("ScrollingFrame")
    TabPage.Name = tabName .. "Page"
    TabPage.Size = UDim2.new(1, 0, 1, 0)
    TabPage.BackgroundTransparency = 1
    TabPage.BorderSizePixel = 0
    TabPage.ScrollBarThickness = 3
    TabPage.AutomaticCanvasSize = Enum.AutomaticSize.Y
    TabPage.CanvasSize = UDim2.new(0, 0, 0, 0)
    TabPage.Visible = false
    TabPage.Parent = ContainerPages

    local PageLayout = Instance.new("UIListLayout")
    PageLayout.SortOrder = Enum.SortOrder.LayoutOrder
    PageLayout.Padding = UDim.new(0, 8)
    PageLayout.Parent = TabPage

    local PagePadding = Instance.new("UIPadding")
    PagePadding.PaddingTop = UDim.new(0, 10)
    PagePadding.PaddingLeft = UDim.new(0, 10)
    PagePadding.PaddingRight = UDim.new(0, 10)
    PagePadding.Parent = TabPage

    TabButton.MouseButton1Click:Connect(function()
        for _, tab in pairs(Tabs) do
            tab.Page.Visible = false
            TweenService:Create(tab.Button, TweenInfo.new(0.2), {
                BackgroundColor3 = Color3.fromRGB(30, 30, 40),
                TextColor3 = Color3.fromRGB(150, 150, 170)
            }):Play()
        end
        TabPage.Visible = true
        TweenService:Create(TabButton, TweenInfo.new(0.2), {
            BackgroundColor3 = activeAccentColor,
            TextColor3 = Color3.fromRGB(255, 255, 255)
        }):Play()
    end)

    table.insert(Tabs, {Button = TabButton, Page = TabPage})
    return TabPage
end

-- =================================================================
-- SELETOR DE JOGADORES (ABA TROLL)
-- =================================================================
local function CreatePlayerDropdown(parentPage)
    local DropdownContainer = Instance.new("Frame")
    DropdownContainer.Size = UDim2.new(1, 0, 0, 38)
    DropdownContainer.BackgroundColor3 = Color3.fromRGB(45, 45, 60)
    DropdownContainer.ZIndex = 5
    DropdownContainer.LayoutOrder = 1
    DropdownContainer.Parent = parentPage

    local DropCorner = Instance.new("UICorner")
    DropCorner.CornerRadius = UDim.new(0, 6)
    DropCorner.Parent = DropdownContainer

    local DropBtn = Instance.new("TextButton")
    DropBtn.Size = UDim2.new(1, 0, 1, 0)
    DropBtn.BackgroundTransparency = 1
    DropBtn.Text = "👤 Selecionar Jogador"
    DropBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    DropBtn.Font = Enum.Font.Gotham
    DropBtn.TextSize = 13
    DropBtn.ZIndex = 6
    DropBtn.Parent = DropdownContainer

    local DropScroll = Instance.new("ScrollingFrame")
    DropScroll.Size = UDim2.new(1, 0, 0, 120)
    DropScroll.Position = UDim2.new(0, 0, 1, 5)
    DropScroll.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    DropScroll.BorderSizePixel = 0
    DropScroll.Visible = false
    DropScroll.ZIndex = 10
    DropScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    DropScroll.ScrollBarThickness = 3
    DropScroll.Parent = DropdownContainer

    local ScrollCorner = Instance.new("UICorner")
    ScrollCorner.CornerRadius = UDim.new(0, 6)
    ScrollCorner.Parent = DropScroll

    local ScrollLayout = Instance.new("UIListLayout")
    ScrollLayout.SortOrder = Enum.SortOrder.LayoutOrder
    ScrollLayout.Padding = UDim.new(0, 4)
    ScrollLayout.Parent = DropScroll

    local function RefreshPlayerList()
        for _, child in ipairs(DropScroll:GetChildren()) do
            if child:IsA("TextButton") then child:Destroy() end
        end
        for _, plr in ipairs(Players:GetPlayers()) do
            local optionBtn = Instance.new("TextButton")
            optionBtn.Size = UDim2.new(1, 0, 0, 28)
            optionBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 65)
            optionBtn.Text = "  👤 " .. plr.DisplayName .. " (@" .. plr.Name .. ")"
            optionBtn.TextColor3 = Color3.fromRGB(220, 220, 220)
            optionBtn.Font = Enum.Font.Gotham
            optionBtn.TextSize = 11
            optionBtn.TextXAlignment = Enum.TextXAlignment.Left
            optionBtn.ZIndex = 11
            optionBtn.Parent = DropScroll

            local btnCorner = Instance.new("UICorner")
            btnCorner.CornerRadius = UDim.new(0, 4)
            btnCorner.Parent = optionBtn

            optionBtn.MouseButton1Click:Connect(function()
                selectedTrollPlayer = plr
                DropBtn.Text = "👤 Jogador: " .. plr.DisplayName
                DropScroll.Visible = false
            end)
        end
    end

    DropBtn.MouseButton1Click:Connect(function()
        RefreshPlayerList()
        DropScroll.Visible = not DropScroll.Visible
    end)
end

-- =================================================================
-- CRIANDO AS ABAS
-- =================================================================
local AbaAdmin = nil
if isOwner then
    AbaAdmin = CreateTab("👑 Admin")
end

local AbaOP = CreateTab("⭐ OP")
local AbaTroll = CreateTab("Troll")
local AbaCasa = CreateTab("🔒 Casa")
local AbaCarro = CreateTab("Carro")
local AbaProtecoes = CreateTab("Proteções")
local AbaConfig = CreateTab("Configurações")

-- --- CONTEÚDO DA ABA ADMIN ---
if isOwner and AbaAdmin then
    local AdminContainer = Instance.new("Frame")
    AdminContainer.Size = UDim2.new(1, 0, 0, 38)
    AdminContainer.BackgroundColor3 = Color3.fromRGB(45, 45, 60)
    AdminContainer.ZIndex = 5
    AdminContainer.LayoutOrder = 1
    AdminContainer.Parent = AbaAdmin
    Instance.new("UICorner", AdminContainer).CornerRadius = UDim.new(0, 6)

    local AdminDropBtn = Instance.new("TextButton")
    AdminDropBtn.Size = UDim2.new(1, 0, 1, 0)
    AdminDropBtn.BackgroundTransparency = 1
    AdminDropBtn.Text = "⚡ Selecionar Jogador Alvo"
    AdminDropBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    AdminDropBtn.Font = Enum.Font.Gotham
    AdminDropBtn.TextSize = 13
    AdminDropBtn.ZIndex = 6
    AdminDropBtn.Parent = AdminContainer

    local AdminDropScroll = Instance.new("ScrollingFrame")
    AdminDropScroll.Size = UDim2.new(1, 0, 0, 120)
    AdminDropScroll.Position = UDim2.new(0, 0, 1, 5)
    AdminDropScroll.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    AdminDropScroll.BorderSizePixel = 0
    AdminDropScroll.Visible = false
    AdminDropScroll.ZIndex = 10
    AdminDropScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
    AdminDropScroll.ScrollBarThickness = 3
    AdminDropScroll.Parent = AdminContainer
    Instance.new("UICorner", AdminDropScroll).CornerRadius = UDim.new(0, 6)

    local AdminScrollLayout = Instance.new("UIListLayout")
    AdminScrollLayout.SortOrder = Enum.SortOrder.LayoutOrder
    AdminScrollLayout.Padding = UDim.new(0, 4)
    AdminScrollLayout.Parent = AdminDropScroll

    local function RefreshAdminPlayerList()
        for _, child in ipairs(AdminDropScroll:GetChildren()) do
            if child:IsA("TextButton") then child:Destroy() end
        end

        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                local optionBtn = Instance.new("TextButton")
                optionBtn.Size = UDim2.new(1, 0, 0, 28)
                optionBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 65)
                optionBtn.Text = "  👤 " .. plr.DisplayName .. " (@" .. plr.Name .. ")"
                optionBtn.TextColor3 = Color3.fromRGB(220, 220, 220)
                optionBtn.Font = Enum.Font.Gotham
                optionBtn.TextSize = 11
                optionBtn.TextXAlignment = Enum.TextXAlignment.Left
                optionBtn.ZIndex = 11
                optionBtn.Parent = AdminDropScroll
                Instance.new("UICorner", optionBtn).CornerRadius = UDim.new(0, 4)

                optionBtn.MouseButton1Click:Connect(function()
                    selectedAdminPlayer = plr
                    AdminDropBtn.Text = "🎯 Alvo: " .. plr.DisplayName
                    AdminDropScroll.Visible = false
                end)
            end
        end
    end

    AdminDropBtn.MouseButton1Click:Connect(function()
        RefreshAdminPlayerList()
        AdminDropScroll.Visible = not AdminDropScroll.Visible
    end)

    local BtnKillUserScript = Instance.new("TextButton")
    BtnKillUserScript.Size = UDim2.new(1, 0, 0, 35)
    BtnKillUserScript.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
    BtnKillUserScript.Text = "💀 Kill Script Jogador"
    BtnKillUserScript.TextColor3 = Color3.fromRGB(255, 255, 255)
    BtnKillUserScript.Font = Enum.Font.GothamBold
    BtnKillUserScript.TextSize = 13
    BtnKillUserScript.LayoutOrder = 2
    BtnKillUserScript.Parent = AbaAdmin
    Instance.new("UICorner", BtnKillUserScript).CornerRadius = UDim.new(0, 6)

    BtnKillUserScript.MouseButton1Click:Connect(function()
        if not selectedAdminPlayer then
            BtnKillUserScript.Text = "❌ Selecione um jogador!"
            task.wait(1.5)
            BtnKillUserScript.Text = "💀 Kill Script Jogador"
            return
        end

        task.spawn(function()
            BtnKillUserScript.Text = "⚡ Destruindo script de " .. selectedAdminPlayer.DisplayName .. "..."
            BtnKillUserScript.BackgroundColor3 = WORKING_BTN_COLOR

            -- Tenta forçar a eliminação e queda do script
            local ok = ExecuteCouchKill(selectedAdminPlayer)

            if ok then
                BtnKillUserScript.Text = "✅ Jogador eliminado!"
                BtnKillUserScript.BackgroundColor3 = ACTIVE_BTN_COLOR
            else
                BtnKillUserScript.Text = "❌ Falha ao eliminar!"
                BtnKillUserScript.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
            end

            task.wait(2)
            BtnKillUserScript.Text = "💀 Kill Script Jogador"
            BtnKillUserScript.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
        end)
    end)
end

-- --- CONTEÚDO DA ABA OP ---
local BtnAllBring = Instance.new("TextButton")
BtnAllBring.Size = UDim2.new(1, 0, 0, 35)
BtnAllBring.BackgroundColor3 = DEFAULT_BTN_COLOR
BtnAllBring.Text = ";all bring"
BtnAllBring.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnAllBring.Font = Enum.Font.GothamBold
BtnAllBring.TextSize = 13
BtnAllBring.LayoutOrder = 1
BtnAllBring.Parent = AbaOP
Instance.new("UICorner", BtnAllBring).CornerRadius = UDim.new(0, 6)

BtnAllBring.MouseButton1Click:Connect(function()
    if allBringRunning then
        allBringRunning = false
        globalActionLock = false
        return
    end

    if globalActionLock then return end

    task.spawn(function()
        allBringRunning = true
        globalActionLock = true
        BtnAllBring.Text = "⏹️ Parar All Bring"
        BtnAllBring.BackgroundColor3 = WORKING_BTN_COLOR

        local previousAntiSitState = antiSitActive
        if not antiSitActive then ToggleAntiSit(true) end

        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        local hum = char and char:FindFirstChildOfClass("Humanoid")

        if not char or not hrp or not hum then
            ToggleAntiSit(previousAntiSitState)
            BtnAllBring.Text = ";all bring"
            BtnAllBring.BackgroundColor3 = DEFAULT_BTN_COLOR
            allBringRunning = false
            globalActionLock = false
            return
        end

        local originalCFrame = hrp.CFrame
        hrp.CFrame = CFrame.new(-82.6, 22.1, -130.0)
        task.wait(0.3)

        if not allBringRunning then
            ToggleAntiSit(previousAntiSitState)
            BtnAllBring.Text = ";all bring"
            BtnAllBring.BackgroundColor3 = DEFAULT_BTN_COLOR
            globalActionLock = false
            return
        end

        hum.Jump = true
        task.wait(0.3)

        local toolName = "Couch"
        local couchTool = char:FindFirstChild(toolName) or (LocalPlayer:FindFirstChild("Backpack") and LocalPlayer.Backpack:FindFirstChild(toolName))
        if couchTool then hum:EquipTool(couchTool) end
        task.wait(0.3)

        local playersToBring = {}
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer then table.insert(playersToBring, p) end
        end

        for _, targetPlayer in ipairs(playersToBring) do
            if not allBringRunning then break end

            local targetChar = targetPlayer.Character
            local targetHrp = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
            local targetHum = targetChar and targetChar:FindFirstChildOfClass("Humanoid")

            if targetChar and targetHrp and targetHum and targetHum.Health > 0 then
                local loopTimeout = 0
                local hasSat = false

                while allBringRunning and loopTimeout < 120 do
                    if not targetChar or not targetHrp or not targetHum or targetHum.Health <= 0 then break end
                    hum.Sit = false
                    hrp.CFrame = targetHrp.CFrame * CFrame.new(0, -3.6, 0)
                    if targetHum.Sit then hasSat = true break end
                    task.wait(0.03)
                    loopTimeout = loopTimeout + 1
                end

                if hasSat and allBringRunning then
                    task.wait(0.1)
                    hrp.CFrame = originalCFrame
                    task.wait(0.5)
                    hum.Jump = true
                    task.wait(0.3)
                end
            end
        end

        clickImage(DELETE_IMG_ID)
        task.wait(0.3)

        if char and hrp then hrp.CFrame = originalCFrame end
        ToggleAntiSit(previousAntiSitState)
        BtnAllBring.Text = "✅ All Bring Concluído!"
        BtnAllBring.BackgroundColor3 = ACTIVE_BTN_COLOR
        task.wait(2)
        BtnAllBring.Text = ";all bring"
        BtnAllBring.BackgroundColor3 = DEFAULT_BTN_COLOR
        allBringRunning = false
        globalActionLock = false
    end)
end)

-- --- CONTEÚDO DA ABA CASA ---
local ConstructionLabel = Instance.new("TextLabel")
ConstructionLabel.Size = UDim2.new(1, 0, 0, 80)
ConstructionLabel.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
ConstructionLabel.Text = "🚧 Aba Casa 🚧\n\nEm construção ainda..."
ConstructionLabel.TextColor3 = Color3.fromRGB(255, 200, 80)
ConstructionLabel.Font = Enum.Font.GothamBold
ConstructionLabel.TextSize = 13
ConstructionLabel.Parent = AbaCasa
Instance.new("UICorner", ConstructionLabel).CornerRadius = UDim.new(0, 8)

-- --- CONTEÚDO DA ABA TROLL ---
CreatePlayerDropdown(AbaTroll)

local BtnView = Instance.new("TextButton")
BtnView.Size = UDim2.new(1, 0, 0, 35)
BtnView.BackgroundColor3 = DEFAULT_BTN_COLOR
BtnView.Text = ";view"
BtnView.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnView.Font = Enum.Font.GothamBold
BtnView.TextSize = 13
BtnView.LayoutOrder = 2
BtnView.Parent = AbaTroll
Instance.new("UICorner", BtnView).CornerRadius = UDim.new(0, 6)

local BtnKill = Instance.new("TextButton")
BtnKill.Size = UDim2.new(1, 0, 0, 35)
BtnKill.BackgroundColor3 = DEFAULT_BTN_COLOR
BtnKill.Text = ";kill"
BtnKill.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnKill.Font = Enum.Font.GothamBold
BtnKill.TextSize = 13
BtnKill.LayoutOrder = 3
BtnKill.Parent = AbaTroll
Instance.new("UICorner", BtnKill).CornerRadius = UDim.new(0, 6)

local BtnBring = Instance.new("TextButton")
BtnBring.Size = UDim2.new(1, 0, 0, 35)
BtnBring.BackgroundColor3 = DEFAULT_BTN_COLOR
BtnBring.Text = ";bring"
BtnBring.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnBring.Font = Enum.Font.GothamBold
BtnBring.TextSize = 13
BtnBring.LayoutOrder = 4
BtnBring.Parent = AbaTroll
Instance.new("UICorner", BtnBring).CornerRadius = UDim.new(0, 6)

local BtnCustomBlock = Instance.new("TextButton")
BtnCustomBlock.Size = UDim2.new(1, 0, 0, 35)
BtnCustomBlock.BackgroundColor3 = DEFAULT_BTN_COLOR
BtnCustomBlock.Text = ";fling"
BtnCustomBlock.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnCustomBlock.Font = Enum.Font.GothamBold
BtnCustomBlock.TextSize = 13
BtnCustomBlock.LayoutOrder = 5
BtnCustomBlock.Parent = AbaTroll
Instance.new("UICorner", BtnCustomBlock).CornerRadius = UDim.new(0, 6)

local BtnAnnoy = Instance.new("TextButton")
BtnAnnoy.Size = UDim2.new(1, 0, 0, 35)
BtnAnnoy.BackgroundColor3 = DEFAULT_BTN_COLOR
BtnAnnoy.Text = ";annoy"
BtnAnnoy.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnAnnoy.Font = Enum.Font.GothamBold
BtnAnnoy.TextSize = 13
BtnAnnoy.LayoutOrder = 6
BtnAnnoy.Parent = AbaTroll
Instance.new("UICorner", BtnAnnoy).CornerRadius = UDim.new(0, 6)

local BtnFlingPorta = Instance.new("TextButton")
BtnFlingPorta.Size = UDim2.new(1, 0, 0, 35)
BtnFlingPorta.BackgroundColor3 = DEFAULT_BTN_COLOR
BtnFlingPorta.Text = ";fling porta"
BtnFlingPorta.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnFlingPorta.Font = Enum.Font.GothamBold
BtnFlingPorta.TextSize = 13
BtnFlingPorta.LayoutOrder = 7
BtnFlingPorta.Parent = AbaTroll
Instance.new("UICorner", BtnFlingPorta).CornerRadius = UDim.new(0, 6)

-- Botão View
BtnView.MouseButton1Click:Connect(function()
    if not selectedTrollPlayer then
        BtnView.Text = "❌ Selecione um jogador!"
        task.wait(1.5)
        BtnView.Text = ";view"
        return
    end
    if viewing then
        stopView()
        BtnView.Text = ";view"
        BtnView.BackgroundColor3 = DEFAULT_BTN_COLOR
    else
        startView(selectedTrollPlayer)
        BtnView.Text = ";view: ON"
        BtnView.BackgroundColor3 = ACTIVE_BTN_COLOR
    end
end)

-- Botão Kill
BtnKill.MouseButton1Click:Connect(function()
    if not selectedTrollPlayer then
        BtnKill.Text = "❌ Selecione um jogador!"
        task.wait(1.5)
        BtnKill.Text = ";kill"
        return
    end

    if globalActionLock then return end

    task.spawn(function()
        BtnKill.Text = "💀 Executando kill..."
        BtnKill.BackgroundColor3 = WORKING_BTN_COLOR

        local ok = ExecuteCouchKill(selectedTrollPlayer)

        if ok then
            BtnKill.Text = "✅ Jogador enviado ao Void!"
            BtnKill.BackgroundColor3 = ACTIVE_BTN_COLOR
        else
            BtnKill.Text = "❌ Falha no Kill!"
            BtnKill.BackgroundColor3 = DEFAULT_BTN_COLOR
        end

        task.wait(2)
        BtnKill.Text = ";kill"
        BtnKill.BackgroundColor3 = DEFAULT_BTN_COLOR
    end)
end)

-- Botão Bring
BtnBring.MouseButton1Click:Connect(function()
    if not selectedTrollPlayer then
        BtnBring.Text = "❌ Selecione um jogador!"
        task.wait(1.5)
        BtnBring.Text = ";bring"
        return
    end

    if globalActionLock then return end

    task.spawn(function()
        BtnBring.Text = "🧲 Executando bring..."
        BtnBring.BackgroundColor3 = WORKING_BTN_COLOR

        local ok = ExecuteCouchBring(selectedTrollPlayer)

        if ok then
            BtnBring.Text = "✅ Jogador trazido com sucesso!"
            BtnBring.BackgroundColor3 = ACTIVE_BTN_COLOR
        else
            BtnBring.Text = "❌ Falha no Bring!"
            BtnBring.BackgroundColor3 = DEFAULT_BTN_COLOR
        end

        task.wait(2)
        BtnBring.Text = ";bring"
        BtnBring.BackgroundColor3 = DEFAULT_BTN_COLOR
    end)
end)

-- Botão Fling
BtnCustomBlock.MouseButton1Click:Connect(function()
    if activeBlockSystem then
        stopBlockSystem()
        BtnCustomBlock.Text = ";fling"
        BtnCustomBlock.BackgroundColor3 = DEFAULT_BTN_COLOR
        BtnAnnoy.Text = ";annoy"
        BtnAnnoy.BackgroundColor3 = DEFAULT_BTN_COLOR
        globalActionLock = false
        return
    end

    if globalActionLock then return end

    task.spawn(function()
        globalActionLock = true
        BtnCustomBlock.Text = "⚽ Preparando bola..."
        BtnCustomBlock.BackgroundColor3 = WORKING_BTN_COLOR
        
        local ok = RunBallSequence()
        if not ok then
            BtnCustomBlock.Text = "❌ Falha ao equipar bola!"
            task.wait(1.5)
            BtnCustomBlock.Text = ";fling"
            BtnCustomBlock.BackgroundColor3 = DEFAULT_BTN_COLOR
            globalActionLock = false
            return
        end

        BtnCustomBlock.Text = "🌪️ Iniciando fling..."
        local success = startBlockSystem("HumanoidRootPart", false, false)
        if success then
            BtnCustomBlock.Text = ";fling: ON"
            BtnCustomBlock.BackgroundColor3 = ACTIVE_BTN_COLOR
        else
            BtnCustomBlock.Text = "❌ Nenhum bloco próximo!"
            task.wait(1.5)
            BtnCustomBlock.Text = ";fling"
            BtnCustomBlock.BackgroundColor3 = DEFAULT_BTN_COLOR
            globalActionLock = false
        end
    end)
end)

-- Botão Annoy
BtnAnnoy.MouseButton1Click:Connect(function()
    if activeBlockSystem then
        stopBlockSystem()
        BtnAnnoy.Text = ";annoy"
        BtnAnnoy.BackgroundColor3 = DEFAULT_BTN_COLOR
        BtnCustomBlock.Text = ";fling"
        BtnCustomBlock.BackgroundColor3 = DEFAULT_BTN_COLOR
        globalActionLock = false
        return
    end

    if globalActionLock then return end

    task.spawn(function()
        globalActionLock = true
        BtnAnnoy.Text = "⚽ Preparando bola..."
        BtnAnnoy.BackgroundColor3 = WORKING_BTN_COLOR

        local ok = RunBallSequence()
        if not ok then
            BtnAnnoy.Text = "❌ Falha ao equipar bola!"
            task.wait(1.5)
            BtnAnnoy.Text = ";annoy"
            BtnAnnoy.BackgroundColor3 = DEFAULT_BTN_COLOR
            globalActionLock = false
            return
        end

        BtnAnnoy.Text = "⚡ Annoy (800 Speed)..."
        local success = startBlockSystem("HumanoidRootPart", false, true)
        if success then
            BtnAnnoy.Text = ";annoy: ON"
            BtnAnnoy.BackgroundColor3 = ACTIVE_BTN_COLOR
        else
            BtnAnnoy.Text = "❌ Nenhum bloco próximo!"
            task.wait(1.5)
            BtnAnnoy.Text = ";annoy"
            BtnAnnoy.BackgroundColor3 = DEFAULT_BTN_COLOR
            globalActionLock = false
        end
    end)
end)

-- Botão Fling Porta
BtnFlingPorta.MouseButton1Click:Connect(function()
    if activeBlockSystem then
        stopBlockSystem()
        BtnFlingPorta.Text = ";fling porta"
        BtnFlingPorta.BackgroundColor3 = DEFAULT_BTN_COLOR
        globalActionLock = false
        return
    end

    if globalActionLock then return end

    globalActionLock = true
    local success = startBlockSystem("HumanoidRootPart", true, false)
    if success then
        BtnFlingPorta.Text = ";fling porta: ON"
        BtnFlingPorta.BackgroundColor3 = ACTIVE_BTN_COLOR
    else
        BtnFlingPorta.Text = "❌ Nenhuma 'Door' física próxima!"
        task.wait(1.5)
        BtnFlingPorta.Text = ";fling porta"
        BtnFlingPorta.BackgroundColor3 = DEFAULT_BTN_COLOR
        globalActionLock = false
    end
end)

-- --- ABA CARRO ---
local function CreateCarToggle(text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 35)
    btn.BackgroundColor3 = DEFAULT_BTN_COLOR
    btn.Text = text .. ": OFF"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    btn.Parent = AbaCarro
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

    local active = false
    btn.MouseButton1Click:Connect(function()
        active = not active
        callback(active, btn)
        if active then
            btn.Text = text .. ": ON"
            btn.BackgroundColor3 = ACTIVE_BTN_COLOR
        else
            btn.Text = text .. ": OFF"
            btn.BackgroundColor3 = DEFAULT_BTN_COLOR
        end
    end)
end

local function CreateCarTextBox(placeholder, defaultVal, onValueChanged)
    local box = Instance.new("TextBox")
    box.Size = UDim2.new(1, 0, 0, 32)
    box.BackgroundColor3 = Color3.fromRGB(45, 45, 60)
    box.Text = tostring(defaultVal)
    box.PlaceholderText = placeholder
    box.TextColor3 = Color3.fromRGB(255, 255, 255)
    box.PlaceholderColor3 = Color3.fromRGB(150, 150, 170)
    box.Font = Enum.Font.Gotham
    box.TextSize = 12
    box.Parent = AbaCarro
    Instance.new("UICorner", box).CornerRadius = UDim.new(0, 6)

    box.FocusLost:Connect(function()
        local num = tonumber(box.Text)
        if num then
            onValueChanged(num)
        else
            box.Text = tostring(defaultVal)
            onValueChanged(defaultVal)
        end
    end)
end

CreateCarToggle("🏎️ Velocidade Carro", function(state) ToggleCarSpeed(state) end)
CreateCarTextBox("Digite a velocidade (ex: 3)", customCarSpeed, function(val) customCarSpeed = val end)

CreateCarToggle("🚀 Turbo Extra", function(state) ToggleCarTurbo(state) end)
CreateCarTextBox("Digite o turbo (ex: 6)", customTurboSpeed, function(val) customTurboSpeed = val end)

-- --- ABA PROTEÇÕES ---
local function CreateProtecaoToggle(text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 35)
    btn.BackgroundColor3 = DEFAULT_BTN_COLOR
    btn.Text = text .. ": OFF"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 12
    btn.Parent = AbaProtecoes
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

    local stateVal = false
    btn.MouseButton1Click:Connect(function()
        stateVal = not stateVal
        callback(stateVal, btn)
        if stateVal then
            btn.Text = text .. ": ON"
            btn.BackgroundColor3 = ACTIVE_BTN_COLOR
        else
            btn.Text = text .. ": OFF"
            btn.BackgroundColor3 = DEFAULT_BTN_COLOR
        end
    end)
end

CreateProtecaoToggle("🛡️ Anti Fling", function(state) ToggleAntiFling(state) end)
CreateProtecaoToggle("🛡️ Anti Kick", function(state) ToggleAntiKick(state) end)
CreateProtecaoToggle("🛡️ Anti Void", function(state) ToggleAntiVoid(state) end)
CreateProtecaoToggle("🛡️ Anti Sit", function(state) ToggleAntiSit(state) end)

-- --- ABA CONFIGURAÇÕES ---
local function CreateConfigBtn(text, color, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 32)
    btn.BackgroundColor3 = color
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.Parent = AbaConfig
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

    btn.MouseButton1Click:Connect(function() callback(btn) end)
    return btn
end

CreateConfigBtn("🖐️ Mover Painel: ATIVADO", Color3.fromRGB(50, 120, 60), function(btn)
    isMainDraggable = not isMainDraggable
    if isMainDraggable then
        btn.Text = "🖐️ Mover Painel: ATIVADO"
        btn.BackgroundColor3 = Color3.fromRGB(50, 120, 60)
    else
        btn.Text = "🖐️ Mover Painel: DESATIVADO"
        btn.BackgroundColor3 = Color3.fromRGB(120, 50, 50)
    end
end)

CreateConfigBtn("❌ Destruir Script (Fechar Tudo)", Color3.fromRGB(150, 30, 30), function()
    ScreenGui:Destroy()
end)

if #Tabs > 0 then
    Tabs[1].Page.Visible = true
    Tabs[1].Button.BackgroundColor3 = activeAccentColor
    Tabs[1].Button.TextColor3 = Color3.fromRGB(255, 255, 255)
end
