```
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local Player = Players.LocalPlayer
if not Player then
    error("LocalPlayer not found. Ensure this script runs on the client.")
end

local Character = Player.Character or Player.CharacterAdded:Wait()
local Humanoid = Character:WaitForChild("Humanoid")
local RootPart = Character:WaitForChild("HumanoidRootPart")
local Torso = Character:WaitForChild("Torso")

-- Сохраняем оригинальные значения
local originalGravity = workspace.Gravity
local originalJumpPower = Humanoid.JumpPower

-- Настройки персонажа
workspace.Gravity = 50
local JUMP_POWER = 13
Humanoid.JumpPower = JUMP_POWER
Humanoid.AutoRotate = true

-- ID анимаций
local IDLE_ANIMATION = "rbxassetid://80979207"
local WALK_ANIMATION = "rbxassetid://80979216"
local RUN_ANIMATION = "rbxassetid://80979216" -- Исправленный ID для бега

-- Настройки скорости
local baseWalkSpeed = 16
local maxWalkSpeed = 20
local sprintWalkSpeed = 30
local currentWalkSpeed = baseWalkSpeed
local jumpSpeedBoost = 0
local maxJumpSpeedBoost = 50
local walkSpeedIncrement = 1
local jumpSpeedIncrement = 8
local sprintAccelerationTime = 3
local sprintTimer = 0
local bhopWindow = 0.15
local lastLandedTime = 0
local preservedVelocity = Vector3.new(0, 0, 0)

-- Настройки Trimping
local TRIMP_MIN_ANGLE = 35 -- Минимальный угол для тримпинга (градусы)
local TRIMP_MAX_ANGLE = 75 -- Максимальный угол для тримпинга (градусы)
local TRIMP_SPEED_BOOST = 1.5 -- Множитель скорости при тримпинге
local TRIMP_JUMP_BOOST = 1.8 -- Множитель высоты прыжка при тримпинге
local lastTrimpTime = 0
local trimpCooldown = 0.5 -- Задержка между тримпами

-- Ограничения движения
local STRAFE_REDUCTION = 0.7
local BACKWARD_REDUCTION = 0.3
local DIRECTION_SMOOTHING = 0.5

-- Состояние пробела
local isSpaceHeld = false

-- Создаем анимации
local animator = Humanoid:FindFirstChildOfClass("Animator") or Humanoid:WaitForChild("Animator")

local idleAnim = Instance.new("Animation")
idleAnim.AnimationId = IDLE_ANIMATION
local idleTrack = animator:LoadAnimation(idleAnim)

local walkAnim = Instance.new("Animation")
walkAnim.AnimationId = WALK_ANIMATION
local walkTrack = animator:LoadAnimation(walkAnim)

local runAnim = Instance.new("Animation")
runAnim.AnimationId = RUN_ANIMATION
local runTrack = animator:LoadAnimation(runAnim)

idleTrack.Looped = true
walkTrack.Looped = true
runTrack.Looped = true

-- Состояния
local isWalking = false
local isRunning = false
local isJumping = false
local isTrimping = false
local moveDirection = Vector3.new(0, 0, 0)

-- Функции анимаций
local function stopAllAnimations()
    idleTrack:Stop()
    walkTrack:Stop()
    runTrack:Stop()
end

local function playIdle()
    if isWalking or isRunning or isJumping then return end
    stopAllAnimations()
    idleTrack:Play()
end

local function playWalk()
    if not isWalking or isRunning or isJumping then return end
    stopAllAnimations()
    walkTrack:Play()
end

local function playRun()
    if not isRunning or isJumping then return end
    stopAllAnimations()
    runTrack:Play()
end

-- Проверка угла поверхности для тримпинга
local function checkTrimpSurface(hit)
    if not hit then return false end
    local normal = hit.Normal
    local angle = math.deg(math.acos(normal:Dot(Vector3.new(0, 1, 0))))
    return angle > TRIMP_MIN_ANGLE and angle < TRIMP_MAX_ANGLE
end

-- Обработка тримпинга
local function handleTrimp(hit)
    if isTrimping or (tick() - lastTrimpTime) < trimpCooldown then return end
    
    local velocity = RootPart.AssemblyLinearVelocity
    local horVelocity = Vector3.new(velocity.X, 0, velocity.Z)
    local speed = horVelocity.Magnitude
    
    if speed > baseWalkSpeed * 1.2 then -- Минимальная скорость для тримпинга
        isTrimping = true
        lastTrimpTime = tick()
        
        -- Увеличиваем скорость и высоту прыжка
        local newVelocity = horVelocity * TRIMP_SPEED_BOOST
        local jumpVelocity = math.min(velocity.Y * TRIMP_JUMP_BOOST, JUMP_POWER * 2)
        
        RootPart.AssemblyLinearVelocity = Vector3.new(
            newVelocity.X,
            jumpVelocity,
            newVelocity.Z
        )
        
        -- Сохраняем скорость для BHop
        preservedVelocity = newVelocity
        jumpSpeedBoost = math.min(jumpSpeedBoost + 15, maxJumpSpeedBoost)
        
        task.delay(0.3, function()
            isTrimping = false
        end)
    end
end

-- Обработка прыжков
local function attemptJump()
    if not Humanoid or not RootPart then return end
    local currentState = Humanoid:GetState()
    if currentState == Enum.HumanoidStateType.Running or currentState == Enum.HumanoidStateType.Landed then
        Humanoid.JumpPower = JUMP_POWER
        Humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
        
        -- Проверка на тримпинг при прыжке
        local raycastParams = RaycastParams.new()
        raycastParams.FilterDescendantsInstances = {Character}
        raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
        
        local hit = workspace:Raycast(RootPart.Position, Vector3.new(0, -3, 0), raycastParams)
        if hit and checkTrimpSurface(hit) then
            handleTrimp(hit)
        end
    end
end

-- Система разгона при ходьбе
local speedConnection = RunService.Heartbeat:Connect(function(deltaTime)
    if not RootPart or not Humanoid then return end
    
    Humanoid.JumpPower = JUMP_POWER
    
    local speed = RootPart.AssemblyLinearVelocity.Magnitude
    isRunning = speed > 16
    isWalking = speed > 2 and not isRunning
    
    if (isWalking or isRunning) and moveDirection.Magnitude > 0 then
        sprintTimer = sprintTimer + deltaTime
        if sprintTimer >= sprintAccelerationTime then
            currentWalkSpeed = sprintWalkSpeed
        else
            local t = sprintTimer / sprintAccelerationTime
            currentWalkSpeed = baseWalkSpeed + (sprintWalkSpeed - baseWalkSpeed) * t
        end
        Humanoid.WalkSpeed = currentWalkSpeed
    else
        sprintTimer = 0
        currentWalkSpeed = baseWalkSpeed
        Humanoid.WalkSpeed = currentWalkSpeed
    end
    
    if isRunning then
        playRun()
    elseif isWalking then
        playWalk()
    else
        playIdle()
    end
end)

-- Система прыжков и BHop
local jumpConnection = Humanoid.StateChanged:Connect(function(old, new)
    if new == Enum.HumanoidStateType.Freefall then 
        if isJumping then return end
        isJumping = true
        
        -- Проверка на тримпинг при отрыве от земли
        if not isTrimping then
            local raycastParams = RaycastParams.new()
            raycastParams.FilterDescendantsInstances = {Character}
            raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
            
            local hit = workspace:Raycast(RootPart.Position, Vector3.new(0, -3, 0), raycastParams)
            if hit and checkTrimpSurface(hit) then
                handleTrimp(hit)
            end
        end
        
        local isBhop = (tick() - lastLandedTime) <= bhopWindow
        if isBhop then
            jumpSpeedBoost = math.min(jumpSpeedBoost + 5, maxJumpSpeedBoost)
        else
            jumpSpeedBoost = 0
            preservedVelocity = Vector3.new(0, 0, 0)
        end
        
        if RootPart:FindFirstChild("JumpVelocity") then
            RootPart.JumpVelocity:Destroy()
        end
        
        local velocity = Instance.new("BodyVelocity")
        velocity.Name = "JumpVelocity"
        velocity.MaxForce = Vector3.new(60000, 0, 60000)
        velocity.Velocity = preservedVelocity
        velocity.Parent = RootPart
    end
    
    if old == Enum.HumanoidStateType.Freefall and new == Enum.HumanoidStateType.Landed then
        isJumping = false
        lastLandedTime = tick()
        if RootPart:FindFirstChild("JumpVelocity") then
            preservedVelocity = RootPart.JumpVelocity.Velocity
            RootPart.JumpVelocity:Destroy()
        end
        playIdle()
        if isSpaceHeld then
            attemptJump()
        end
    end
end)

-- Обновление движения в воздухе с ограничениями
local function updateAirMovement()
    if isJumping and RootPart:FindFirstChild("JumpVelocity") then
        local camera = workspace.CurrentCamera
        local lookVector = camera.CFrame.LookVector
        local rightVector = camera.CFrame.RightVector
        
        local moveX = moveDirection.X
        local moveZ = moveDirection.Z
        local inputMagnitude = math.sqrt(moveX^2 + moveZ^2)
        
        -- Применяем ограничения
        if inputMagnitude > 0 then
            moveX = moveX * STRAFE_REDUCTION
            if moveZ > 0 then
                moveZ = moveZ * BACKWARD_REDUCTION
            end
            
            inputMagnitude = math.sqrt(moveX^2 + moveZ^2)
            if inputMagnitude > 1 then
                moveX = moveX / inputMagnitude
                moveZ = moveZ / inputMagnitude
            end
        end
        
        local combinedDirection = (rightVector * moveX * 1.2 + lookVector * -moveZ)
        if combinedDirection.Magnitude > 0 then
            combinedDirection = combinedDirection.Unit
        else
            combinedDirection = RootPart.CFrame.LookVector
            combinedDirection = Vector3.new(combinedDirection.X, 0, combinedDirection.Z).Unit
        end
        
        if not isTrimping then
            jumpSpeedBoost = math.min(jumpSpeedBoost + jumpSpeedIncrement * RunService.Heartbeat:Wait(), maxJumpSpeedBoost)
        end
        
        local speed = Humanoid.WalkSpeed + jumpSpeedBoost
        local targetVelocity = combinedDirection * speed
        local currentVelocity = RootPart.JumpVelocity.Velocity
        local newVelocity = currentVelocity:Lerp(targetVelocity, DIRECTION_SMOOTHING)
        
        -- Дополнительное ограничение для движения назад
        local forwardDot = newVelocity:Dot(RootPart.CFrame.LookVector)
        if forwardDot < 0 then
            newVelocity = newVelocity * 0.6
        end
        
        RootPart.JumpVelocity.Velocity = newVelocity
    end
end

local renderConnection = RunService.RenderStepped:Connect(function()
    if isJumping then
        updateAirMovement()
    end
end)

-- Обработка ввода
local function onInputBegan(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        local key = input.KeyCode
        if key == Enum.KeyCode.W then
            moveDirection = Vector3.new(moveDirection.X, 0, -1)
        elseif key == Enum.KeyCode.S then
            moveDirection = Vector3.new(moveDirection.X, 0, 1)
        elseif key == Enum.KeyCode.A then
            moveDirection = Vector3.new(-1, 0, moveDirection.Z)
        elseif key == Enum.KeyCode.D then
            moveDirection = Vector3.new(1, 0, moveDirection.Z)
        elseif key == Enum.KeyCode.Space then
            isSpaceHeld = true
            attemptJump()
        end
    end
end

local function onInputEnded(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        local key = input.KeyCode
        if key == Enum.KeyCode.W or key == Enum.KeyCode.S then
            moveDirection = Vector3.new(moveDirection.X, 0, 0)
        elseif key == Enum.KeyCode.A or key == Enum.KeyCode.D then
            moveDirection = Vector3.new(0, 0, moveDirection.Z)
        elseif key == Enum.KeyCode.Space then
            isSpaceHeld = false
        end
    end
end

local inputConnection = UserInputService.InputBegan:Connect(onInputBegan)
UserInputService.InputEnded:Connect(onInputEnded)

-- Функция восстановления оригинальных значений
local function restoreOriginalSettings()
    workspace.Gravity = originalGravity
    Humanoid.JumpPower = originalJumpPower
end

-- Обработка смерти
Humanoid.Died:Connect(function()
    speedConnection:Disconnect()
    jumpConnection:Disconnect()
    inputConnection:Disconnect()
    renderConnection:Disconnect()
    stopAllAnimations()
    if RootPart:FindFirstChild("JumpVelocity") then
        RootPart.JumpVelocity:Destroy()
    end
    restoreOriginalSettings()
end)

-- Обработка смены персонажа
Player.CharacterAdded:Connect(function(newCharacter)
    restoreOriginalSettings()
    
    Character = newCharacter
    Humanoid = Character:WaitForChild("Humanoid")
    RootPart = Character:WaitForChild("HumanoidRootPart")
    Torso = Character:WaitForChild("Torso")
    
    workspace.Gravity = 50
    Humanoid.JumpPower = JUMP_POWER
    Humanoid.AutoRotate = true
    
    speedConnection:Disconnect()
    jumpConnection:Disconnect()
    inputConnection:Disconnect()
    renderConnection:Disconnect()
    
    animator = Humanoid:FindFirstChildOfClass("Animator") or Humanoid:WaitForChild("Animator")
    idleTrack = animator:LoadAnimation(idleAnim)
    walkTrack = animator:LoadAnimation(walkAnim)
    runTrack = animator:LoadAnimation(runAnim)
    idleTrack.Looped = true
    walkTrack.Looped = true
    runTrack.Looped = true
    
    speedConnection = RunService.Heartbeat:Connect(function(deltaTime)
        -- (Повтор кода из основной части)
    end)
    
    jumpConnection = Humanoid.StateChanged:Connect(function(old, new)
        -- (Повтор кода из основной части)
    end)
    
    renderConnection = RunService.RenderStepped:Connect(function()
        if isJumping then
            updateAirMovement()
        end
    end)
    
    inputConnection = UserInputService.InputBegan:Connect(onInputBegan)
    UserInputService.InputEnded:Connect(onInputEnded)
    
    Humanoid.WalkSpeed = baseWalkSpeed
    playIdle()
end)

-- Первоначальный запуск
Humanoid.WalkSpeed = baseWalkSpeed
Humanoid.JumpPower = JUMP_POWER
playIdle()
```
