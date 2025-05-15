```
-- Настройки
local defaultFlySpeed = 1 -- Базовая скорость (1 как в IY)
local guiColor = Color3.fromRGB(0, 170, 255)
local turnSpeed = 5 -- Скорость поворота
local toggleKey = Enum.KeyCode.F

-- Создание GUI
local Player = game:GetService("Players").LocalPlayer
local Camera = workspace.CurrentCamera
local UserInputService = game:GetService("UserInputService")

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "FlyGUI"
ScreenGui.Parent = Player.PlayerGui
ScreenGui.ResetOnSpawn = false

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, 200, 0, 50)
Frame.Position = UDim2.new(0.5, -100, 0.1, 0)
Frame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
Frame.BackgroundTransparency = 0.3
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true
Frame.Parent = ScreenGui

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(0, 8)
Corner.Parent = Frame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 20)
Title.Position = UDim2.new(0, 0, 0, 5)
Title.BackgroundTransparency = 1
Title.Text = "Flight Control"
Title.TextColor3 = guiColor
Title.Font = Enum.Font.GothamBold
Title.TextSize = 14
Title.Parent = Frame

local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1, 0, 0, 20)
Status.Position = UDim2.new(0, 0, 0, 25)
Status.BackgroundTransparency = 1
Status.Text = "Status: OFF"
Status.TextColor3 = Color3.fromRGB(255, 50, 50)
Status.Font = Enum.Font.Gotham
Status.TextSize = 12
Status.Parent = Frame

-- Логика полёта
local flying = false
local currentFlySpeed = defaultFlySpeed
local bodyGyro
local bodyVelocity
local lastLookVector = Vector3.new(0, 0, 1)

local function startFlying(speed)
    if flying then 
        stopFlying()
        task.wait(0.1)
    end
    
    currentFlySpeed = speed or defaultFlySpeed
    
    local character = Player.Character
    if not character then
        character = Player.CharacterAdded:Wait()
        task.wait(0.5) -- Даем время на загрузку персонажа
    end
    
    local humanoid = character:WaitForChild("Humanoid")
    humanoid.PlatformStand = true
    
    bodyGyro = Instance.new("BodyGyro")
    bodyGyro.P = 10000
    bodyGyro.maxTorque = Vector3.new(100000, 100000, 100000)
    bodyGyro.cframe = CFrame.new(character.HumanoidRootPart.Position, character.HumanoidRootPart.Position + lastLookVector)
    bodyGyro.Parent = character.HumanoidRootPart
    
    bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.Velocity = Vector3.new(0, 0, 0)
    bodyVelocity.maxForce = Vector3.new(10000, 10000, 10000)
    bodyVelocity.Parent = character.HumanoidRootPart
    
    flying = true
    Status.Text = "Status: ON (Speed: "..currentFlySpeed..")"
    Status.TextColor3 = Color3.fromRGB(50, 255, 50)
end

local function stopFlying()
    if not flying then return end
    
    local character = Player.Character
    if character then
        local humanoid = character:FindFirstChild("Humanoid")
        if humanoid then
            humanoid.PlatformStand = false
        end
        
        if character:FindFirstChild("HumanoidRootPart") then
            for _, v in pairs(character.HumanoidRootPart:GetChildren()) do
                if v:IsA("BodyGyro") or v:IsA("BodyVelocity") then
                    v:Destroy()
                end
            end
        end
    end
    
    flying = false
    Status.Text = "Status: OFF"
    Status.TextColor3 = Color3.fromRGB(255, 50, 50)
end

local function updateFlight()
    if not flying then return end
    
    local character = Player.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    
    local rootPart = character.HumanoidRootPart
    local cameraLookVector = Camera.CFrame.LookVector
    lastLookVector = lastLookVector:Lerp(cameraLookVector, turnSpeed * task.wait())
    
    -- Обновляем только поворот
    bodyGyro.cframe = CFrame.new(rootPart.Position, rootPart.Position + lastLookVector)
    
    -- Рассчитываем движение
    local moveDirection = Vector3.new(0, 0, 0)
    local speedMultiplier = currentFlySpeed * 50
    
    -- Получаем базовые направления
    local forward = lastLookVector
    local right = rootPart.CFrame.RightVector
    local up = Vector3.new(0, 1, 0)
    
    -- Убираем вертикальную составляющую из вектора вперед
    forward = Vector3.new(forward.X, 0, forward.Z).Unit
    
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then
        moveDirection = moveDirection + (forward * speedMultiplier)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then
        moveDirection = moveDirection - (forward * speedMultiplier)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then
        moveDirection = moveDirection - (right * speedMultiplier)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then
        moveDirection = moveDirection + (right * speedMultiplier)
    end
    
    -- Вертикальное движение
    if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
        moveDirection = moveDirection + (up * speedMultiplier)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
        moveDirection = moveDirection - (up * speedMultiplier)
    end
    
    -- Применяем движение
    bodyVelocity.Velocity = moveDirection
end

-- Надежная система команд через чат
local function setupChatCommands()
    -- Обработчик команд
    local function handleCommand(message)
        message = message:lower()
        
        if message:sub(1, 5) == ".fly " then
            local speed = tonumber(message:sub(6)) or defaultFlySpeed
            startFlying(math.clamp(speed, 0.1, 10))
        elseif message == ".fly" then
            startFlying()
        elseif message == ".unfly" then
            stopFlying()
        end
    end
    
    -- Подключение к разным системам чата
    local function connectToChat()
        -- Стандартная система чата Roblox
        if game:GetService("ReplicatedStorage"):FindFirstChild("DefaultChatSystemChatEvents") then
            local chatEvents = game:GetService("ReplicatedStorage").DefaultChatSystemChatEvents
            local sayMessage = chatEvents:FindFirstChild("SayMessageRequest") or chatEvents:FindFirstChild("SayMessageRequest")
            if sayMessage then
                sayMessage.OnClientEvent:Connect(function(msg)
                    handleCommand(msg)
                end)
            end
        end
        
        -- Резервный метод
        Player.Chatted:Connect(handleCommand)
    end
    
    -- Задержка для инициализации чата
    task.spawn(function()
        connectToChat()
    end)
end

-- Инициализация команд
setupChatCommands()

-- Обработка клавиши F
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.KeyCode == toggleKey then
        if flying then
            stopFlying()
        else
            startFlying(1) -- Всегда скорость 1 при активации на F
        end
    end
end)

-- Основной цикл
game:GetService("RunService").Heartbeat:Connect(function()
    updateFlight()
end)

-- Очистка при выходе
Player.CharacterRemoving:Connect(function()
    stopFlying()
end)

-- Автоматический перезапуск при смерти
Player.CharacterAdded:Connect(function()
    if flying then
        task.wait(1) -- Даем время на появление персонажа
        startFlying(currentFlySpeed)
    end
end)

local function callback(Text)
end
 
local NotificationBindable = Instance.new("BindableFunction")
NotificationBindable.OnInvoke = callback
 
game.StarterGui:SetCore("SendNotification", {
    Title = "Flight Control v1";
    Text = "Pls sub in my TG: @hhh_best | F to toggle!";
    Duration = "5";
    Callback = NotificationBindable;
})
```
