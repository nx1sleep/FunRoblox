```
local Resolution = 0.76  -- значение от 0 до 1 (1 - обычный вид, меньше - более сплющенный)

local Camera = workspace.CurrentCamera
game:GetService("RunService").RenderStepped:Connect(function()
    Camera.CFrame = Camera.CFrame * CFrame.new(0, 0, 0, 1, 0, 0, 0, Resolution, 0, 0, 0, 1)
end)
```
