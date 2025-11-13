-- Frostfall UI Library v2.0
-- Premium Dark Luxury UI with Snowflake Effects
-- Single-script, loadstring-ready, 1000+ lines

local FrostfallUI = {}
FrostfallUI.__index = FrostfallUI

-- ===== CONFIGURATION =====
local Config = {
    -- Color Scheme: Deep Red + Black + White
    Colors = {
        Primary = Color3.fromRGB(80, 0, 0),        -- Deep burgundy
        Secondary = Color3.fromRGB(40, 0, 0),      -- Dark red
        Accent = Color3.fromRGB(255, 255, 255),    -- Pure white
        Background = Color3.fromRGB(10, 0, 0),     -- Near-black with red tint
        Text = Color3.fromRGB(255, 255, 255),      -- White text
        Stroke = Color3.fromRGB(120, 0, 0)         -- Red stroke
    },
    
    -- Visual Effects
    BlurIntensity = 0.7,
    SnowflakeCount = 80,
    AnimationSpeed = 0.4,
    
    -- UI Dimensions
    WindowSize = UDim2.new(0, 550, 0, 450),
    TabSize = UDim2.new(0, 140, 0, 35),
    ElementSize = UDim2.new(1, -20, 0, 35),
    
    -- Snowflake Properties
    Snowflake = {
        MinSize = 8,
        MaxSize = 18,
        MinSpeed = 25,
        MaxSpeed = 45,
        MinTransparency = 0.3,
        MaxTransparency = 0.7,
        SpinRange = 5,
        WindEffect = 0.5
    }
}

-- ===== SERVICES =====
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")

-- ===== UTILITIES =====
local function CreateInstance(className, properties)
    local instance = Instance.new(className)
    for property, value in pairs(properties) do
        if property ~= "Parent" then
            instance[property] = value
        end
    end
    if properties.Parent then
        instance.Parent = properties.Parent
    end
    return instance
end

local function CreateTween(object, properties, duration, easingStyle, easingDirection)
    local tweenInfo = TweenInfo.new(
        duration or Config.AnimationSpeed,
        easingStyle or Enum.EasingStyle.Quad,
        easingDirection or Enum.EasingDirection.Out
    )
    local tween = TweenService:Create(object, tweenInfo, properties)
    tween:Play()
    return tween
end

local function RoundCorners(object, radius)
    CreateInstance("UICorner", {
        CornerRadius = UDim.new(0, radius or 6),
        Parent = object
    })
end

local function AddStroke(object, color, thickness)
    CreateInstance("UIStroke", {
        Color = color or Config.Colors.Stroke,
        Thickness = thickness or 1,
        Parent = object
    })
end

-- ===== SNOWFLAKE SYSTEM =====
local SnowSystem = {}
SnowSystem.__index = SnowSystem

function SnowSystem.new(parent)
    local self = setmetatable({}, SnowSystem)
    self.Parent = parent
    self.Snowflakes = {}
    self.Connections = {}
    self:Initialize()
    return self
end

function SnowSystem:Initialize()
    -- Create layered containers for parallax effect
    self.BackgroundLayer = CreateInstance("Frame", {
        Name = "SnowBackground",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 1, 0),
        ZIndex = 1,
        Parent = self.Parent
    })
    
    self.MidgroundLayer = CreateInstance("Frame", {
        Name = "SnowMidground",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 1, 0),
        ZIndex = 2,
        Parent = self.Parent
    })
    
    self.ForegroundLayer = CreateInstance("Frame", {
        Name = "SnowForeground",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 1, 0),
        ZIndex = 3,
        Parent = self.Parent
    })
    
    -- Distribute snowflakes across layers
    local layers = {self.BackgroundLayer, self.MidgroundLayer, self.ForegroundLayer}
    local flakesPerLayer = math.floor(Config.SnowflakeCount / 3)
    
    for layerIndex, layer in ipairs(layers) do
        for i = 1, flakesPerLayer do
            self:CreateSnowflake(layer, layerIndex, (layerIndex - 1) * flakesPerLayer + i)
        end
    end
    
    -- Animation loop
    self.Connections.Render = RunService.RenderStepped:Connect(function(deltaTime)
        self:UpdateSnowflakes(deltaTime)
    end)
end

function SnowSystem:CreateSnowflake(layer, layerIndex, id)
    local size = math.random(Config.Snowflake.MinSize, Config.Snowflake.MaxSize)
    local speed = math.random(Config.Snowflake.MinSpeed, Config.Snowflake.MaxSpeed)
    local transparency = math.random(
        Config.Snowflake.MinTransparency * 100,
        Config.Snowflake.MaxTransparency * 100
    ) / 100
    
    -- Layer-based properties for parallax
    local layerMultipliers = {0.7, 1.0, 1.3} -- Background, Midground, Foreground
    local speedMultiplier = layerMultipliers[layerIndex]
    local sizeMultiplier = layerMultipliers[layerIndex]
    local windMultiplier = layerMultipliers[layerIndex]
    
    local snowflake = CreateInstance("ImageLabel", {
        Name = "Snowflake_" .. id,
        BackgroundTransparency = 1,
        Size = UDim2.new(0, size * sizeMultiplier, 0, size * sizeMultiplier),
        Position = UDim2.new(0, math.random(-100, 1400), 0, math.random(-200, -50)),
        Image = "rbxassetid://9751992395", -- High-quality snowflake
        ImageColor3 = Config.Colors.Accent,
        ImageTransparency = transparency,
        Rotation = math.random(0, 360),
        ZIndex = layer.ZIndex,
        Parent = layer
    })
    
    RoundCorners(snowflake, 100)
    
    self.Snowflakes[id] = {
        Object = snowflake,
        BaseSpeed = speed * speedMultiplier,
        SpinSpeed = math.random(-Config.Snowflake.SpinRange, Config.Snowflake.SpinRange),
        Amplitude = math.random(15, 40) * windMultiplier,
        Frequency = math.random(5, 15) * 0.01,
        TimeOffset = math.random(0, 100),
        Layer = layerIndex
    }
end

function SnowSystem:UpdateSnowflakes(deltaTime)
    local screenSize = self.Parent.AbsoluteSize
    
    for id, snowflake in pairs(self.Snowflakes) do
        if snowflake.Object and snowflake.Object.Parent then
            local currentPos = snowflake.Object.Position
            local currentRotation = snowflake.Object.Rotation
            
            -- Wind effect with sine wave
            local windOffset = math.sin((tick() + snowflake.TimeOffset) * snowflake.Frequency) * snowflake.Amplitude
            local newX = currentPos.X.Offset + windOffset * deltaTime
            local newY = currentPos.Y.Offset + snowflake.BaseSpeed * deltaTime
            local newRotation = currentRotation + snowflake.SpinSpeed * deltaTime
            
            -- Reset if out of bounds
            if newY > screenSize.Y + 100 then
                newY = -50
                newX = math.random(-100, screenSize.X + 100)
            end
            
            if newX < -150 or newX > screenSize.X + 150 then
                newX = math.random(-100, screenSize.X + 100)
            end
            
            snowflake.Object.Position = UDim2.new(0, newX, 0, newY)
            snowflake.Object.Rotation = newRotation
        end
    end
end

function SnowSystem:Destroy()
    for _, connection in pairs(self.Connections) do
        connection:Disconnect()
    end
    
    for _, snowflake in pairs(self.Snowflakes) do
        if snowflake.Object then
            snowflake.Object:Destroy()
        end
    end
    
    if self.BackgroundLayer then self.BackgroundLayer:Destroy() end
    if self.MidgroundLayer then self.MidgroundLayer:Destroy() end
    if self.ForegroundLayer then self.ForegroundLayer:Destroy() end
end

-- ===== BASE UI ELEMENT =====
local UIElement = {}
UIElement.__index = UIElement

function UIElement.new(name, parent)
    local self = setmetatable({}, UIElement)
    self.Name = name
    self.Parent = parent
    self.Visible = true
    self.Enabled = true
    return self
end

function UIElement:Show()
    if self.Container then
        self.Container.Visible = true
        self.Visible = true
    end
end

function UIElement:Hide()
    if self.Container then
        self.Container.Visible = false
        self.Visible = false
    end
end

function UIElement:Enable()
    self.Enabled = true
    if self.Container then
        self.Container.Active = true
        self.Container.BackgroundTransparency = 0.1
    end
end

function UIElement:Disable()
    self.Enabled = false
    if self.Container then
        self.Container.Active = false
        self.Container.BackgroundTransparency = 0.5
    end
end

function UIElement:Destroy()
    if self.Container then
        self.Container:Destroy()
    end
end

-- ===== BUTTON COMPONENT =====
local Button = setmetatable({}, UIElement)
Button.__index = Button

function Button.new(name, parent, text, callback)
    local self = setmetatable(UIElement.new(name, parent), Button)
    self.Text = text
    self.Callback = callback
    self:Create()
    return self
end

function Button:Create()
    self.Container = CreateInstance("Frame", {
        Name = self.Name,
        BackgroundColor3 = Config.Colors.Primary,
        BackgroundTransparency = 0.1,
        Size = Config.ElementSize,
        Parent = self.Parent
    })
    
    RoundCorners(self.Container, 4)
    AddStroke(self.Container, Config.Colors.Stroke, 1)
    
    local button = CreateInstance("TextButton", {
        Name = "Button",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 1, 0),
        Text = "",
        Parent = self.Container
    })
    
    local label = CreateInstance("TextLabel", {
        Name = "Label",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, -10, 1, 0),
        Position = UDim2.new(0, 5, 0, 0),
        Text = self.Text,
        TextColor3 = Config.Colors.Text,
        TextSize = 14,
        TextXAlignment = Enum.TextXAlignment.Left,
        Font = Enum.Font.GothamSemibold,
        Parent = self.Container
    })
    
    local icon = CreateInstance("TextLabel", {
        Name = "Icon",
        BackgroundTransparency = 1,
        Size = UDim2.new(0, 20, 0, 20),
        Position = UDim2.new(1, -25, 0.5, -10),
        Text = "→",
        TextColor3 = Config.Colors.Accent,
        TextSize = 16,
        Font = Enum.Font.GothamBold,
        Parent = self.Container
    })
    
    -- Interactive effects
    button.MouseEnter:Connect(function()
        if self.Enabled then
            CreateTween(self.Container, {BackgroundTransparency = 0.05}, 0.2)
            CreateTween(icon, {TextColor3 = Color3.fromRGB(200, 200, 255)}, 0.2)
        end
    end)
    
    button.MouseLeave:Connect(function()
        if self.Enabled then
            CreateTween(self.Container, {BackgroundTransparency = 0.1}, 0.2)
            CreateTween(icon, {TextColor3 = Config.Colors.Accent}, 0.2)
        end
    end)
    
    button.MouseButton1Down:Connect(function()
        if self.Enabled then
            CreateTween(self.Container, {BackgroundTransparency = 0.2}, 0.1)
        end
    end)
    
    button.MouseButton1Up:Connect(function()
        if self.Enabled then
            CreateTween(self.Container, {BackgroundTransparency = 0.05}, 0.1)
        end
    end)
    
    button.MouseButton1Click:Connect(function()
        if self.Enabled and self.Callback then
            self.Callback()
        end
    end)
end

-- ===== TOGGLE COMPONENT =====
local Toggle = setmetatable({}, UIElement)
Toggle.__index = Toggle

function Toggle.new(name, parent, text, default, callback)
    local self = setmetatable(UIElement.new(name, parent), Toggle)
    self.Text = text
    self.State = default or false
    self.Callback = callback
    self:Create()
    return self
end

function Toggle:Create()
    self.Container = CreateInstance("Frame", {
        Name = self.Name,
        BackgroundTransparency = 1,
        Size = Config.ElementSize,
        Parent = self.Parent
    })
    
    local label = CreateInstance("TextLabel", {
        Name = "Label",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, -70, 1, 0),
        Position = UDim2.new(0, 0, 0, 0),
        Text = self.Text,
        TextColor3 = Config.Colors.Text,
        TextSize = 14,
        TextXAlignment = Enum.TextXAlignment.Left,
        Font = Enum.Font.Gotham,
        Parent = self.Container
    })
    
    self.ToggleFrame = CreateInstance("Frame", {
        Name = "ToggleFrame",
        BackgroundColor3 = Config.Colors.Secondary,
        BackgroundTransparency = 0.2,
        Size = UDim2.new(0, 60, 0, 30),
        Position = UDim2.new(1, -60, 0.5, -15),
        Parent = self.Container
    })
    
    RoundCorners(self.ToggleFrame, 15)
    AddStroke(self.ToggleFrame, Config.Colors.Stroke, 1)
    
    self.ToggleButton = CreateInstance("Frame", {
        Name = "ToggleButton",
        BackgroundColor3 = self.State and Config.Colors.Accent or Color3.fromRGB(100, 100, 100),
        Size = UDim2.new(0, 26, 0, 26),
        Position = self.State and UDim2.new(1, -28, 0.5, -13) or UDim2.new(0, 2, 0.5, -13),
        Parent = self.ToggleFrame
    })
    
    RoundCorners(self.ToggleButton, 13)
    
    local button = CreateInstance("TextButton", {
        Name = "Button",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 1, 0),
        Text = "",
        Parent = self.Container
    })
    
    self:UpdateVisual()
    
    button.MouseButton1Click:Connect(function()
        if self.Enabled then
            self:SetState(not self.State)
        end
    end)
end

function Toggle:SetState(state)
    self.State = state
    self:UpdateVisual()
    
    if self.Callback then
        self.Callback(self.State)
    end
end

function Toggle:UpdateVisual()
    if self.State then
        CreateTween(self.ToggleButton, {
            Position = UDim2.new(1, -28, 0.5, -13),
            BackgroundColor3 = Config.Colors.Accent
        }, 0.2)
        CreateTween(self.ToggleFrame, {
            BackgroundColor3 = Config.Colors.Primary
        }, 0.2)
    else
        CreateTween(self.ToggleButton, {
            Position = UDim2.new(0, 2, 0.5, -13),
            BackgroundColor3 = Color3.fromRGB(100, 100, 100)
        }, 0.2)
        CreateTween(self.ToggleFrame, {
            BackgroundColor3 = Config.Colors.Secondary
        }, 0.2)
    end
end

-- ===== SLIDER COMPONENT =====
local Slider = setmetatable({}, UIElement)
Slider.__index = Slider

function Slider.new(name, parent, text, min, max, default, callback)
    local self = setmetatable(UIElement.new(name, parent), Slider)
    self.Text = text
    self.Min = min or 0
    self.Max = max or 100
    self.Value = default or math.floor((max - min) / 2)
    self.Callback = callback
    self.Dragging = false
    self:Create()
    return self
end

function Slider:Create()
    self.Container = CreateInstance("Frame", {
        Name = self.Name,
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 0, 70),
        Parent = self.Parent
    })
    
    self.Label = CreateInstance("TextLabel", {
        Name = "Label",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 0, 20),
        Text = self.Text .. ": " .. self.Value,
        TextColor3 = Config.Colors.Text,
        TextSize = 14,
        TextXAlignment = Enum.TextXAlignment.Left,
        Font = Enum.Font.Gotham,
        Parent = self.Container
    })
    
    self.ValueLabel = CreateInstance("TextLabel", {
        Name = "ValueLabel",
        BackgroundTransparency = 1,
        Size = UDim2.new(0, 60, 0, 20),
        Position = UDim2.new(1, -60, 0, 0),
        Text = tostring(self.Value),
        TextColor3 = Config.Colors.Accent,
        TextSize = 14,
        TextXAlignment = Enum.TextXAlignment.Right,
        Font = Enum.Font.GothamSemibold,
        Parent = self.Container
    })
    
    self.SliderBackground = CreateInstance("Frame", {
        Name = "SliderBackground",
        BackgroundColor3 = Config.Colors.Secondary,
        BackgroundTransparency = 0.2,
        Size = UDim2.new(1, 0, 0, 20),
        Position = UDim2.new(0, 0, 0, 35),
        Parent = self.Container
    })
    
    RoundCorners(self.SliderBackground, 4)
    AddStroke(self.SliderBackground, Config.Colors.Stroke, 1)
    
    self.Fill = CreateInstance("Frame", {
        Name = "Fill",
        BackgroundColor3 = Config.Colors.Accent,
        Size = UDim2.new((self.Value - self.Min) / (self.Max - self.Min), 0, 1, 0),
        Parent = self.SliderBackground
    })
    
    RoundCorners(self.Fill, 4)
    
    self.SliderButton = CreateInstance("TextButton", {
        Name = "SliderButton",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 1, 0),
        Text = "",
        Parent = self.SliderBackground
    })
    
    self:SetupDrag()
end

function Slider:SetupDrag()
    self.SliderButton.MouseButton1Down:Connect(function()
        if self.Enabled then
            self.Dragging = true
            self:UpdateValueFromMouse()
        end
    end)
    
    local inputEnded
    inputEnded = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            self.Dragging = false
        end
    end)
    
    self.SliderButton.MouseMoved:Connect(function()
        if self.Dragging and self.Enabled then
            self:UpdateValueFromMouse()
        end
    end)
end

function Slider:UpdateValueFromMouse()
    local absolutePos = self.SliderBackground.AbsolutePosition
    local absoluteSize = self.SliderBackground.AbsoluteSize
    local mousePos = UserInputService:GetMouseLocation()
    
    local relativeX = math.clamp((mousePos.X - absolutePos.X) / absoluteSize.X, 0, 1)
    local rawValue = self.Min + (relativeX * (self.Max - self.Min))
    local steppedValue = math.floor(rawValue)
    
    self:SetValue(steppedValue)
end

function Slider:SetValue(value)
    local clampedValue = math.clamp(value, self.Min, self.Max)
    
    if clampedValue ~= self.Value then
        self.Value = clampedValue
        self:UpdateVisual()
        
        if self.Callback then
            self.Callback(self.Value)
        end
    end
end

function Slider:UpdateVisual()
    local fillRatio = (self.Value - self.Min) / (self.Max - self.Min)
    
    CreateTween(self.Fill, {Size = UDim2.new(fillRatio, 0, 1, 0)}, 0.1)
    self.Label.Text = self.Text .. ": " .. self.Value
    self.ValueLabel.Text = tostring(self.Value)
end

-- ===== DROPDOWN COMPONENT =====
local Dropdown = setmetatable({}, UIElement)
Dropdown.__index = Dropdown

function Dropdown.new(name, parent, text, options, default, callback)
    local self = setmetatable(UIElement.new(name, parent), Dropdown)
    self.Text = text
    self.Options = options or {}
    self.Selected = default or (options[1] or "Select...")
    self.Callback = callback
    self.Expanded = false
    self:Create()
    return self
end

function Dropdown:Create()
    self.Container = CreateInstance("Frame", {
        Name = self.Name,
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 0, 35),
        Parent = self.Parent,
        ClipsDescendants = true
    })
    
    self.MainButton = CreateInstance("TextButton", {
        Name = "MainButton",
        BackgroundColor3 = Config.Colors.Primary,
        BackgroundTransparency = 0.1,
        Size = UDim2.new(1, 0, 0, 35),
        Text = "",
        Parent = self.Container
    })
    
    RoundCorners(self.MainButton, 4)
    AddStroke(self.MainButton, Config.Colors.Stroke, 1)
    
    CreateInstance("TextLabel", {
        Name = "Label",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, -60, 1, 0),
        Position = UDim2.new(0, 10, 0, 0),
        Text = self.Text .. ": " .. self.Selected,
        TextColor3 = Config.Colors.Text,
        TextSize = 14,
        TextXAlignment = Enum.TextXAlignment.Left,
        Font = Enum.Font.Gotham,
        Parent = self.MainButton
    })
    
    self.Arrow = CreateInstance("TextLabel", {
        Name = "Arrow",
        BackgroundTransparency = 1,
        Size = UDim2.new(0, 20, 0, 20),
        Position = UDim2.new(1, -25, 0.5, -10),
        Text = "▼",
        TextColor3 = Config.Colors.Accent,
        TextSize = 12,
        Font = Enum.Font.GothamBold,
        Parent = self.MainButton
    })
    
    self.OptionsFrame = CreateInstance("Frame", {
        Name = "OptionsFrame",
        BackgroundColor3 = Config.Colors.Secondary,
        BackgroundTransparency = 0.1,
        Size = UDim2.new(1, 0, 0, 0),
        Position = UDim2.new(0, 0, 0, 40),
        Parent = self.Container,
        ClipsDescendants = true
    })
    
    RoundCorners(self.OptionsFrame, 4)
    AddStroke(self.OptionsFrame, Config.Colors.Stroke, 1)
    
    CreateInstance("UIListLayout", {
        Parent = self.OptionsFrame,
        SortOrder = Enum.SortOrder.LayoutOrder
    })
    
    self:CreateOptions()
    
    self.MainButton.MouseButton1Click:Connect(function()
        if self.Enabled then
            self:Toggle()
        end
    end)
end

function Dropdown:CreateOptions()
    for _, child in pairs(self.OptionsFrame:GetChildren()) do
        if child:IsA("TextButton") then
            child:Destroy()
        end
    end
    
    for i, option in ipairs(self.Options) do
        local optionButton = CreateInstance("TextButton", {
            Name = "Option_" .. option,
            BackgroundColor3 = Config.Colors.Primary,
            BackgroundTransparency = 0.3,
            Size = UDim2.new(1, 0, 0, 30),
            Text = option,
            TextColor3 = Config.Colors.Text,
            TextSize = 12,
            Font = Enum.Font.Gotham,
            LayoutOrder = i,
            Parent = self.OptionsFrame
        })
        
        optionButton.MouseButton1Click:Connect(function()
            self:Select(option)
        end)
        
        optionButton.MouseEnter:Connect(function()
            CreateTween(optionButton, {BackgroundTransparency = 0.1}, 0.2)
        end)
        
        optionButton.MouseLeave:Connect(function()
            CreateTween(optionButton, {BackgroundTransparency = 0.3}, 0.2)
        end)
    end
end

function Dropdown:Toggle()
    self.Expanded = not self.Expanded
    
    if self.Expanded then
        local optionCount = #self.Options
        local maxHeight = math.min(optionCount * 30 + 10, 150)
        CreateTween(self.OptionsFrame, {Size = UDim2.new(1, 0, 0, maxHeight)}, 0.3)
        CreateTween(self.Container, {Size = UDim2.new(1, 0, 0, 40 + maxHeight)}, 0.3)
        CreateTween(self.Arrow, {Rotation = 180}, 0.3)
    else
        CreateTween(self.OptionsFrame, {Size = UDim2.new(1, 0, 0, 0)}, 0.3)
        CreateTween(self.Container, {Size = UDim2.new(1, 0, 0, 35)}, 0.3)
        CreateTween(self.Arrow, {Rotation = 0}, 0.3)
    end
end

function Dropdown:Select(option)
    self.Selected = option
    self.MainButton:FindFirstChild("Label").Text = self.Text .. ": " .. option
    self:Toggle()
    
    if self.Callback then
        self.Callback(option)
    end
end

-- ===== LABEL COMPONENT =====
local Label = setmetatable({}, UIElement)
Label.__index = Label

function Label.new(name, parent, text)
    local self = setmetatable(UIElement.new(name, parent), Label)
    self.Text = text
    self:Create()
    return self
end

function Label:Create()
    self.Container = CreateInstance("Frame", {
        Name = self.Name,
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 0, 25),
        Parent = self.Parent
    })
    
    CreateInstance("TextLabel", {
        Name = "Label",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 1, 0),
        Text = self.Text,
        TextColor3 = Config.Colors.Text,
        TextSize = 14,
        TextXAlignment = Enum.TextXAlignment.Left,
        Font = Enum.Font.Gotham,
        Parent = self.Container
    })
end

-- ===== TAB SYSTEM =====
local Tab = setmetatable({}, UIElement)
Tab.__index = Tab

function Tab.new(name, parent, text)
    local self = setmetatable(UIElement.new(name, parent), Tab)
    self.Text = text
    self.Elements = {}
    self:Create()
    return self
end

function Tab:Create()
    self.Container = CreateInstance("Frame", {
        Name = self.Name,
        BackgroundTransparency = 1,
        Size = UDim2.new(1, -150, 1, -50),
        Position = UDim2.new(0, 140, 0, 40),
        Visible = false,
        Parent = self.Parent
    })
    
    CreateInstance("UIListLayout", {
        Parent = self.Container,
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 8)
    })
    
    CreateInstance("UIPadding", {
        Parent = self.Container,
        PaddingLeft = UDim.new(0, 10),
        PaddingRight = UDim.new(0, 10),
        PaddingTop = UDim.new(0, 10),
        PaddingBottom = UDim.new(0, 10)
    })
end

function Tab:AddButton(text, callback)
    local button = Button.new("Button_" .. text, self.Container, text, callback)
    table.insert(self.Elements, button)
    return self
end

function Tab:AddToggle(text, default, callback)
    local toggle = Toggle.new("Toggle_" .. text, self.Container, text, default, callback)
    table.insert(self.Elements, toggle)
    return self
end

function Tab:AddSlider(text, min, max, default, callback)
    local slider = Slider.new("Slider_" .. text, self.Container, text, min, max, default, callback)
    table.insert(self.Elements, slider)
    return self
end

function Tab:AddDropdown(text, options, default, callback)
    local dropdown = Dropdown.new("Dropdown_" .. text, self.Container, text, options, default, callback)
    table.insert(self.Elements, dropdown)
    return self
end

function Tab:AddLabel(text)
    local label = Label.new("Label_" .. text, self.Container, text)
    table.insert(self.Elements, label)
    return self
end

-- ===== MAIN UI CLASS =====
function FrostfallUI.new(title)
    local self = setmetatable({}, FrostfallUI)
    self.Title = title or "Frostfall UI"
    self.Tabs = {}
    self.CurrentTab = nil
    self.IsOpen = false
    self:Initialize()
    return self
end

function FrostfallUI:Initialize()
    if self.IsOpen then return end
    
    -- Cleanup existing UI
    local playerGui = Players.LocalPlayer:WaitForChild("PlayerGui")
    local existingUI = playerGui:FindFirstChild("FrostfallUI")
    if existingUI then
        existingUI:Destroy()
    end
    
    -- Create main screen GUI
    self.ScreenGui = CreateInstance("ScreenGui", {
        Name = "FrostfallUI",
        DisplayOrder = 999,
        Parent = playerGui
    })
    
    -- Create dark blur overlay
    self.Overlay = CreateInstance("Frame", {
        Name = "Overlay",
        BackgroundColor3 = Config.Colors.Background,
        BackgroundTransparency = Config.BlurIntensity,
        Size = UDim2.new(1, 0, 1, 0),
        Parent = self.ScreenGui
    })
    
    -- Initialize snowflake system
    self.SnowSystem = SnowSystem.new(self.Overlay)
    
    -- Create main window
    self.MainFrame = CreateInstance("Frame", {
        Name = "MainFrame",
        BackgroundColor3 = Config.Colors.Primary,
        BackgroundTransparency = 0.1,
        Size = Config.WindowSize,
        Position = UDim2.new(0.5, -Config.WindowSize.X.Offset/2, 0.5, -Config.WindowSize.Y.Offset/2),
        AnchorPoint = Vector2.new(0.5, 0.5),
        Parent = self.ScreenGui
    })
    
    RoundCorners(self.MainFrame, 12)
    AddStroke(self.MainFrame, Config.Colors.Stroke, 2)
    
    -- Create title bar
    local titleBar = CreateInstance("Frame", {
        Name = "TitleBar",
        BackgroundColor3 = Config.Colors.Secondary,
        BackgroundTransparency = 0.05,
        Size = UDim2.new(1, 0, 0, 40),
        Parent = self.MainFrame
    })
    
    RoundCorners(titleBar, 12)
    
    CreateInstance("TextLabel", {
        Name = "TitleLabel",
        BackgroundTransparency = 1,
        Size = UDim2.new(1, -80, 1, 0),
        Position = UDim2.new(0, 15, 0, 0),
        Text = self.Title,
        TextColor3 = Config.Colors.Text,
        TextSize = 18,
        TextXAlignment = Enum.TextXAlignment.Left,
        Font = Enum.Font.GothamBold,
        Parent = titleBar
    })
    
    -- Close button
    local closeButton = CreateInstance("TextButton", {
        Name = "CloseButton",
        BackgroundColor3 = Config.Colors.Accent,
        BackgroundTransparency = 0.8,
        Size = UDim2.new(0, 25, 0, 25),
        Position = UDim2.new(1, -30, 0.5, -12.5),
        Text = "×",
        TextColor3 = Config.Colors.Text,
        TextSize = 20,
        Font = Enum.Font.GothamBold,
        Parent = titleBar
    })
    
    RoundCorners(closeButton, 100)
    
    closeButton.MouseButton1Click:Connect(function()
        self:Close()
    end)
    
    -- Create tab container
    self.TabContainer = CreateInstance("Frame", {
        Name = "TabContainer",
        BackgroundTransparency = 1,
        Size = UDim2.new(0, 130, 1, -50),
        Position = UDim2.new(0, 5, 0, 45),
        Parent = self.MainFrame
    })
    
    CreateInstance("UIListLayout", {
        Parent = self.TabContainer,
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 5)
    })
    
    -- Make window draggable
    self:MakeDraggable(titleBar)
    
    -- Setup keybinds
    self:SetupKeybinds()
    
    -- Initial animation
    self:OpenAnimation()
    
    self.IsOpen = true
    
    print("Frostfall UI Loaded! | Toggle: RightControl")
end

function FrostfallUI:MakeDraggable(frame)
    local dragging = false
    local dragInput, dragStart, startPos
    
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = self.MainFrame.Position
            
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    
    frame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            dragInput = input
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            self.MainFrame.Position = UDim2.new(
                startPos.X.Scale, 
                startPos.X.Offset + delta.X,
                startPos.Y.Scale, 
                startPos.Y.Offset + delta.Y
            )
        end
    end)
end

function FrostfallUI:SetupKeybinds()
    self.KeyConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.KeyCode == Enum.KeyCode.RightControl then
            self:Toggle()
        end
    end)
end

function FrostfallUI:OpenAnimation()
    self.MainFrame.Size = UDim2.new(0, 10, 0, 10)
    self.MainFrame.BackgroundTransparency = 1
    self.Overlay.BackgroundTransparency = 1
    
    CreateTween(self.Overlay, {BackgroundTransparency = Config.BlurIntensity}, Config.AnimationSpeed)
    CreateTween(self.MainFrame, {
        Size = Config.WindowSize, 
        BackgroundTransparency = 0.1
    }, Config.AnimationSpeed, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
end

function FrostfallUI:CloseAnimation(callback)
    CreateTween(self.MainFrame, {
        Size = UDim2.new(0, 10, 0, 10), 
        BackgroundTransparency = 1
    }, Config.AnimationSpeed)
    CreateTween(self.Overlay, {BackgroundTransparency = 1}, Config.AnimationSpeed)
    
    if callback then
        delay(Config.AnimationSpeed, callback)
    end
end

function FrostfallUI:Close()
    if not self.IsOpen then return end
    
    self:CloseAnimation(function()
        if self.SnowSystem then
            self.SnowSystem:Destroy()
        end
        if self.ScreenGui then
            self.ScreenGui:Destroy()
        end
        if self.KeyConnection then
            self.KeyConnection:Disconnect()
        end
        self.IsOpen = false
    end)
end

function FrostfallUI:Toggle()
    if self.IsOpen then
        self:Close()
    else
        self:Initialize()
    end
end

function FrostfallUI:AddTab(name)
    if not self.IsOpen then
        warn("UI must be open to add tabs. Call :Initialize() first.")
        return
    end
    
    local tab = Tab.new("Tab_" .. name, self.MainFrame, name)
    table.insert(self.Tabs, tab)
    
    -- Create tab button
    local tabButton = CreateInstance("TextButton", {
        Name = "TabButton_" .. name,
        BackgroundColor3 = Config.Colors.Secondary,
        BackgroundTransparency = 0.3,
        Size = Config.TabSize,
        Text = name,
        TextColor3 = Config.Colors.Text,
        TextSize = 13,
        Font = Enum.Font.GothamSemibold,
        Parent = self.TabContainer
    })
    
    RoundCorners(tabButton, 6)
    AddStroke(tabButton, Config.Colors.Stroke, 1)
    
    tabButton.MouseButton1Click:Connect(function()
        self:SwitchTab(tab)
    end)
    
    -- Set as current tab if first tab
    if #self.Tabs == 1 then
        self:SwitchTab(tab)
    end
    
    return tab
end

function FrostfallUI:SwitchTab(tab)
    if not self.IsOpen then return end
    
    -- Hide current tab
    if self.CurrentTab then
        self.CurrentTab:Hide()
    end
    
    -- Show new tab
    tab:Show()
    self.CurrentTab = tab
    
    -- Update tab button appearances
    for _, existingTab in pairs(self.Tabs) do
        local tabButton = self.TabContainer:FindFirstChild("TabButton_" .. existingTab.Text)
        if tabButton then
            if existingTab == tab then
                CreateTween(tabButton, {BackgroundTransparency = 0.1}, 0.2)
                CreateTween(tabButton, {BackgroundColor3 = Config.Colors.Primary}, 0.2)
            else
                CreateTween(tabButton, {BackgroundTransparency = 0.3}, 0.2)
                CreateTween(tabButton, {BackgroundColor3 = Config.Colors.Secondary}, 0.2)
            end
        end
    end
end

-- ===== PUBLIC API =====
local function CreateUI(title)
    return FrostfallUI.new(title)
end

return CreateUI
