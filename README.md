--[[ 
    Ultimate Cheat UI v9.1
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local Camera = Workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- ■ 設定 (Config) ■
local Config = {
    -- Aimbot
    Aimbot_Enabled = false,
    Aimbot_Strength = 100, 
    AimFOV_Enabled = false, 
    AimFOV_Radius = 120,    
    TeamCheck = false,
    WallCheck = false,    -- [NEW] 壁判定
    TouchTrigger = false, 
    AimUI_Move = false, 
    
    -- ESP
    ESP_Box = false,
    ESP_Skeleton = false,
    ESP_Line = false, 
    ESP_Name = false,
    ESP_HP = false,
    ESP_Dist = false,
    ESP_Face = false,
    ESP_Item = false,
    ESP_Radar = false,
    Radar_Move = false,
    
    -- Player
    WalkSpeed_Enabled = false,
    WalkSpeed_Value = 16,
    JumpPower_Enabled = false,
    JumpPower_Value = 50,
    FOV_Enabled = false,
    FOV_Value = 70,
    Crosshair_Enabled = false,
    Crosshair_Size = 12,
    Noclip_Enabled = false,
    InfJump_Enabled = false,
    
    -- World
    ClearFog = false,
    Brightness_Enabled = false,
    Brightness_Value = 0,
    
    -- Settings
    Color_ESP_Skeleton = Color3.fromRGB(255, 255, 255),
    Color_ESP_Box = Color3.fromRGB(255, 0, 0),
    Color_ESP_Line = Color3.fromRGB(255, 255, 255),
    Color_Crosshair = Color3.fromRGB(0, 255, 0),
    Color_UI_Theme = Color3.fromRGB(0, 170, 255),
    Player_Skin_Size = 0,
    Menu_Move = false,
}

local State = {
    CurrentTab = nil,
    Aiming = false, 
    Target = nil,
    ESP_Storage = {},
    Radar_Blips = {}
}

-- ■ UI 初期化 ■
local ParentGui = LocalPlayer:WaitForChild("PlayerGui")
for _, g in pairs(ParentGui:GetChildren()) do
    if g.Name == "UltimateCheatUI_v9" then g:Destroy() end
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "UltimateCheatUI_v9"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = ParentGui
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

-- ■ Drawing API チェック ■
local DrawingApi = Drawing
if not DrawingApi then
    DrawingApi = {new = function() return {Remove = function() end, Destroy = function() end} end} 
end

-- ■ FOV Circle ■
local FOVCircle = DrawingApi.new("Circle")
FOVCircle.Visible = false
FOVCircle.Color = Color3.new(1, 0, 0)
FOVCircle.Thickness = 2
FOVCircle.Radius = 120
FOVCircle.Transparency = 0.6
FOVCircle.Filled = false

-- ■ Crosshair (Reticle) Objects ■
local CrosshairH = DrawingApi.new("Line")
local CrosshairV = DrawingApi.new("Line")
CrosshairH.Visible = false
CrosshairV.Visible = false
CrosshairH.Thickness = 1.5
CrosshairV.Thickness = 1.5
CrosshairH.Color = Config.Color_Crosshair
CrosshairV.Color = Config.Color_Crosshair


-- ■ Helper Functions ■
local function CreateCorner(parent, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius)
    c.Parent = parent
end

local function IsInputInFrame(input, frame)
    if not frame.Visible then return false end
    local pos = input.Position
    local absPos = frame.AbsolutePosition
    local absSize = frame.AbsoluteSize
    
    return pos.X >= absPos.X and pos.X <= absPos.X + absSize.X and
           pos.Y >= absPos.Y and pos.Y <= absPos.Y + absSize.Y
end

-- [NEW] Wall Check Raycast Function
local function IsVisible(targetPart, targetChar)
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    
    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {LocalPlayer.Character, Camera} -- 自分とカメラを無視
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.IgnoreWater = true
    
    local result = Workspace:Raycast(origin, direction, params)
    
    -- 何にも当たらなければ見える
    if not result then return true end
    
    -- 当たったものがターゲットの一部なら見える
    if result.Instance:IsDescendantOf(targetChar) then return true end
    
    -- 壁に当たった
    return false
end

-- ■ MENU UI ■
local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Name = "MenuBtn"
ToggleBtn.Parent = ScreenGui
ToggleBtn.Size = UDim2.new(0, 50, 0, 50)
ToggleBtn.Position = UDim2.new(0, 10, 0.3, 0)
ToggleBtn.BackgroundColor3 = Config.Color_UI_Theme
ToggleBtn.Text = "MENU"
ToggleBtn.TextColor3 = Color3.new(1,1,1)
ToggleBtn.Font = Enum.Font.GothamBold
ToggleBtn.TextSize = 12
CreateCorner(ToggleBtn, 12)

local menuDragging = false
local menuDragStart, menuStartPos

ToggleBtn.InputBegan:Connect(function(input)
    if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1) then
        if Config.Menu_Move then
            menuDragging = true
            menuDragStart = input.Position
            menuStartPos = ToggleBtn.Position
        end
    end
end)

ToggleBtn.InputChanged:Connect(function(input)
    if menuDragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
        local delta = input.Position - menuDragStart
        ToggleBtn.Position = UDim2.new(
            menuStartPos.X.Scale, menuStartPos.X.Offset + delta.X,
            menuStartPos.Y.Scale, menuStartPos.Y.Offset + delta.Y
        )
    end
end)

ToggleBtn.InputEnded:Connect(function(input)
    if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1) then
        menuDragging = false
    end
end)

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.Size = UDim2.new(0, 450, 0, 280) 
MainFrame.Position = UDim2.new(0.5, -225, 0.5, -140)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MainFrame.Visible = false
MainFrame.Active = true
MainFrame.Draggable = true
CreateCorner(MainFrame, 10)

local SideBar = Instance.new("ScrollingFrame")
SideBar.Parent = MainFrame
SideBar.Size = UDim2.new(0, 110, 1, -20)
SideBar.Position = UDim2.new(0, 10, 0, 10)
SideBar.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
SideBar.ScrollBarThickness = 0
CreateCorner(SideBar, 8)

local SideLayout = Instance.new("UIListLayout")
SideLayout.Parent = SideBar
SideLayout.Padding = UDim.new(0, 5)
SideLayout.SortOrder = Enum.SortOrder.LayoutOrder

local ContentFrame = Instance.new("Frame")
ContentFrame.Parent = MainFrame
ContentFrame.Size = UDim2.new(1, -140, 1, -20)
ContentFrame.Position = UDim2.new(0, 130, 0, 10)
ContentFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
CreateCorner(ContentFrame, 8)

local CloseBtn = Instance.new("TextButton")
CloseBtn.Parent = MainFrame
CloseBtn.Size = UDim2.new(0, 24, 0, 24)
CloseBtn.Position = UDim2.new(1, -12, 0, -12)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.new(1,1,1)
CreateCorner(CloseBtn, 12)

ToggleBtn.Activated:Connect(function() 
    if not menuDragging then 
        MainFrame.Visible = not MainFrame.Visible 
    end
end)
CloseBtn.Activated:Connect(function() MainFrame.Visible = false end)

local Tabs = {}
local function SwitchTab(tabName)
    for name, frame in pairs(Tabs) do frame.Visible = (name == tabName) end
end

local function CreateTab(name)
    local btn = Instance.new("TextButton")
    btn.Parent = SideBar
    btn.Size = UDim2.new(1, 0, 0, 40)
    btn.BackgroundColor3 = Color3.fromRGB(40, 40, 45)
    btn.Text = name
    btn.TextColor3 = Color3.new(0.8, 0.8, 0.8)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 14
    CreateCorner(btn, 6)
    
    local page = Instance.new("ScrollingFrame")
    page.Parent = ContentFrame
    page.Size = UDim2.new(1, 0, 1, 0)
    page.BackgroundTransparency = 1
    page.ScrollBarThickness = 4
    page.Visible = false
    
    local list = Instance.new("UIListLayout")
    list.Parent = page
    list.Padding = UDim.new(0, 8)
    list.SortOrder = Enum.SortOrder.LayoutOrder
    
    local pad = Instance.new("UIPadding")
    pad.Parent = page
    pad.PaddingTop = UDim.new(0, 5)
    pad.PaddingLeft = UDim.new(0, 5)
    
    btn.Activated:Connect(function() SwitchTab(name) end)
    Tabs[name] = page
    return page
end

local function AddToggle(page, text, flag, callback)
    local frame = Instance.new("Frame")
    frame.Parent = page
    frame.Size = UDim2.new(1, -10, 0, 35)
    frame.BackgroundColor3 = Color3.fromRGB(45, 45, 50)
    CreateCorner(frame, 6)
    
    local btn = Instance.new("TextButton")
    btn.Parent = frame
    btn.Size = UDim2.new(1, 0, 1, 0)
    btn.BackgroundTransparency = 1
    btn.Text = "  " .. text
    btn.TextColor3 = Color3.new(1,1,1)
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.Font = Enum.Font.GothamSemibold
    btn.TextSize = 13
    
    local status = Instance.new("Frame")
    status.Parent = frame
    status.Size = UDim2.new(0, 10, 0, 10)
    status.Position = UDim2.new(1, -20, 0.5, -5)
    status.BackgroundColor3 = Config[flag] and Config.Color_UI_Theme or Color3.fromRGB(80, 80, 80)
    CreateCorner(status, 10)
    
    btn.Activated:Connect(function()
        Config[flag] = not Config[flag]
        status.BackgroundColor3 = Config[flag] and Config.Color_UI_Theme or Color3.fromRGB(80, 80, 80)
        if callback then callback(Config[flag]) end
    end)
end

local function AddSlider(page, text, flag, min, max, callback)
    local frame = Instance.new("Frame")
    frame.Parent = page
    frame.Size = UDim2.new(1, -10, 0, 50)
    frame.BackgroundColor3 = Color3.fromRGB(45, 45, 50)
    CreateCorner(frame, 6)
    
    local label = Instance.new("TextLabel")
    label.Parent = frame
    label.Size = UDim2.new(1, -10, 0, 20)
    label.Position = UDim2.new(0, 5, 0, 2)
    label.BackgroundTransparency = 1
    label.Text = text .. ": " .. Config[flag]
    label.TextColor3 = Color3.new(1,1,1)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Font = Enum.Font.Gotham
    label.TextSize = 12
    
    local slideBg = Instance.new("TextButton") 
    slideBg.Parent = frame
    slideBg.Size = UDim2.new(1, -20, 0, 8)
    slideBg.Position = UDim2.new(0, 10, 0.65, 0)
    slideBg.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    slideBg.Text = ""
    slideBg.AutoButtonColor = false
    CreateCorner(slideBg, 4)
    
    local fill = Instance.new("Frame")
    fill.Parent = slideBg
    local pct = (Config[flag] - min) / (max - min)
    fill.Size = UDim2.new(math.clamp(pct, 0, 1), 0, 1, 0)
    fill.BackgroundColor3 = Config.Color_UI_Theme
    CreateCorner(fill, 4)
    
    local function Update(input)
        local posX = input.Position.X
        local frameX = slideBg.AbsolutePosition.X
        local sizeX = slideBg.AbsoluteSize.X
        local p = math.clamp((posX - frameX) / sizeX, 0, 1)
        local val = math.floor(min + ((max - min) * p))
        
        Config[flag] = val
        label.Text = text .. ": " .. val
        fill.Size = UDim2.new(p, 0, 1, 0)
        if callback then callback(val) end
    end
    
    local dragging = false
    slideBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            Update(input)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
            Update(input)
        end
    end)
end

local function AddButton(page, text, callback)
    local btn = Instance.new("TextButton")
    btn.Parent = page
    btn.Size = UDim2.new(1, -10, 0, 35)
    btn.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
    btn.Text = text
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 13
    CreateCorner(btn, 6)
    btn.Activated:Connect(callback)
end

-- ■ Aim Button ■
local AimButton = Instance.new("TextButton")
AimButton.Name = "MobileAimTrigger"
AimButton.Parent = ScreenGui
AimButton.Size = UDim2.new(0, 60, 0, 60)
AimButton.Position = UDim2.new(0.8, 0, 0.6, 0)
AimButton.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
AimButton.BackgroundTransparency = 0.5
AimButton.Text = "AIM"
AimButton.TextColor3 = Color3.new(1,1,1)
AimButton.Visible = false 
AimButton.Active = false 
AimButton.Selectable = false
CreateCorner(AimButton, 30)
Instance.new("UIStroke", AimButton).Color = Color3.new(1,1,1)

local aimBtnDragging = false
local aimBtnDragStart, aimBtnStartPos
local aimInputObject = nil

UserInputService.InputBegan:Connect(function(input)
    if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1) then
        if AimButton.Visible and IsInputInFrame(input, AimButton) then
            if Config.AimUI_Move then
                aimBtnDragging = true
                aimBtnDragStart = input.Position
                aimBtnStartPos = AimButton.Position
            else
                aimInputObject = input
                State.Aiming = true
                AimButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
                AimButton.BackgroundTransparency = 0.2
            end
        end
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if aimBtnDragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
        local delta = input.Position - aimBtnDragStart
        AimButton.Position = UDim2.new(
            aimBtnStartPos.X.Scale, aimBtnStartPos.X.Offset + delta.X,
            aimBtnStartPos.Y.Scale, aimBtnStartPos.Y.Offset + delta.Y
        )
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input == aimInputObject then
        State.Aiming = false
        aimInputObject = nil
        AimButton.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
        AimButton.BackgroundTransparency = 0.5
    end
    if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1) then
        aimBtnDragging = false
    end
end)


-- ■ Radar ■
local RadarFrame = Instance.new("Frame")
RadarFrame.Name = "Radar"
RadarFrame.Parent = ScreenGui
RadarFrame.Size = UDim2.new(0, 120, 0, 120)
RadarFrame.Position = UDim2.new(0, 10, 0, 50) 
RadarFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
RadarFrame.BackgroundTransparency = 0.5
RadarFrame.Visible = false
CreateCorner(RadarFrame, 60)
local RadarBorder = Instance.new("UIStroke", RadarFrame)
RadarBorder.Color = Color3.new(1,1,1)
RadarBorder.Thickness = 2
RadarFrame.Active = false 

local RadarCenter = Instance.new("Frame", RadarFrame)
RadarCenter.Size = UDim2.new(0, 6, 0, 6)
RadarCenter.Position = UDim2.new(0.5, -3, 0.5, -3)
RadarCenter.BackgroundColor3 = Color3.new(1,1,1)
CreateCorner(RadarCenter, 3)

local radarDragging = false
local radarDragStart, radarStartPos

UserInputService.InputBegan:Connect(function(input)
    if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1) then
        if Config.Radar_Move and RadarFrame.Visible and IsInputInFrame(input, RadarFrame) then
            radarDragging = true
            radarDragStart = input.Position
            radarStartPos = RadarFrame.Position
        end
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if radarDragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
        local delta = input.Position - radarDragStart
        RadarFrame.Position = UDim2.new(
            radarStartPos.X.Scale, radarStartPos.X.Offset + delta.X,
            radarStartPos.Y.Scale, radarStartPos.Y.Offset + delta.Y
        )
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        radarDragging = false
    end
end)


local function UpdateRadar()
    if Config.Radar_Move then
        RadarFrame.Visible = true
        RadarFrame.BackgroundTransparency = 0.2
    else
        RadarFrame.BackgroundTransparency = 0.5
        if not Config.ESP_Radar then 
            RadarFrame.Visible = false 
            return 
        end
        RadarFrame.Visible = true
    end
    
    local center = Vector2.new(60, 60)
    local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    
    if not myRoot then return end
    
    for p, blip in pairs(State.Radar_Blips) do
        if not p.Parent then blip:Destroy(); State.Radar_Blips[p] = nil end
    end
    
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
            local enemyRoot = plr.Character.HumanoidRootPart
            local look = myRoot.CFrame.LookVector
            local right = myRoot.CFrame.RightVector
            local diff = enemyRoot.Position - myRoot.Position
            
            local distX = diff:Dot(right)
            local distZ = diff:Dot(look)
            local range = 80
            
            local blipX = math.clamp((distX / range) * 60, -60, 60)
            local blipY = math.clamp(-(distZ / range) * 60, -60, 60) 
            
            if not State.Radar_Blips[plr] then
                local b = Instance.new("Frame", RadarFrame)
                b.Size = UDim2.new(0, 4, 0, 4)
                b.BackgroundColor3 = Color3.new(1,0,0)
                CreateCorner(b, 2)
                State.Radar_Blips[plr] = b
            end
            
            local blip = State.Radar_Blips[plr]
            if (Vector2.new(blipX, blipY).Magnitude <= 60) then
                blip.Visible = true
                blip.Position = UDim2.new(0.5, blipX - 2, 0.5, blipY - 2)
            else
                blip.Visible = false
            end
        else
            if State.Radar_Blips[plr] then State.Radar_Blips[plr].Visible = false end
        end
    end
end


-- ■ UI ページ構成 ■

-- 1. Aimbot
local Tab_Aim = CreateTab("Aimbot")
AddToggle(Tab_Aim, "Aimbot Enabled", "Aimbot_Enabled")
AddSlider(Tab_Aim, "Strength (Speed)", "Aimbot_Strength", 1, 100)
AddToggle(Tab_Aim, "Show FOV Circle", "AimFOV_Enabled") 
AddSlider(Tab_Aim, "FOV Radius", "AimFOV_Radius", 10, 500, function(v) FOVCircle.Radius = v end)
AddToggle(Tab_Aim, "Team Check", "TeamCheck")
AddToggle(Tab_Aim, "Wall Check", "WallCheck") -- [NEW]
AddToggle(Tab_Aim, "Touch Only (Btn)", "TouchTrigger", function(v) AimButton.Visible = v end)
AddToggle(Tab_Aim, "Move Aim Button", "AimUI_Move")

-- 2. ESP
local Tab_ESP = CreateTab("ESP")
AddToggle(Tab_ESP, "Box ESP", "ESP_Box")
AddToggle(Tab_ESP, "Skeleton (Lines)", "ESP_Skeleton")
AddToggle(Tab_ESP, "Tracers (Lines)", "ESP_Line") 
AddToggle(Tab_ESP, "Name Tags", "ESP_Name")
AddToggle(Tab_ESP, "Health", "ESP_HP")
AddToggle(Tab_ESP, "Distance", "ESP_Dist")
AddToggle(Tab_ESP, "Face Icon", "ESP_Face")
AddToggle(Tab_ESP, "Item (Tool) Icon", "ESP_Item")
AddToggle(Tab_ESP, "Radar Map", "ESP_Radar")
AddToggle(Tab_ESP, "Move Radar (Edit)", "Radar_Move")

-- 3. Player
local Tab_Plr = CreateTab("Player")
AddToggle(Tab_Plr, "No Clip (Walls)", "Noclip_Enabled")
AddToggle(Tab_Plr, "Infinite Jump", "InfJump_Enabled")
AddToggle(Tab_Plr, "Crosshair Enabled", "Crosshair_Enabled")
AddSlider(Tab_Plr, "Crosshair Size", "Crosshair_Size", 4, 50)
AddToggle(Tab_Plr, "Speed Enabled", "WalkSpeed_Enabled")
AddSlider(Tab_Plr, "WalkSpeed", "WalkSpeed_Value", 0, 200)
AddToggle(Tab_Plr, "Jump Enabled", "JumpPower_Enabled")
AddSlider(Tab_Plr, "JumpPower", "JumpPower_Value", 0, 300)
AddToggle(Tab_Plr, "Cam FOV Enabled", "FOV_Enabled")
AddSlider(Tab_Plr, "Camera FOV", "FOV_Value", 0, 120, function(v) Camera.FieldOfView = v end)

-- 4. World
local Tab_Wld = CreateTab("World")
AddButton(Tab_Wld, "Server Hop", function()
    local Http = game:GetService("HttpService")
    local Servers = Http:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"))
    for _, s in ipairs(Servers.data) do
        if s.playing < s.maxPlayers and s.id ~= game.JobId then
            TeleportService:TeleportToPlaceInstance(game.PlaceId, s.id, LocalPlayer)
            break
        end
    end
end)
AddButton(Tab_Wld, "Rejoin Server", function()
    TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, LocalPlayer)
end)
AddToggle(Tab_Wld, "Clear Fog", "ClearFog", function(v)
    if v then Lighting.FogEnd = 100000 else Lighting.FogEnd = 500 end 
end)
AddToggle(Tab_Wld, "Custom Brightness", "Brightness_Enabled")
AddSlider(Tab_Wld, "Brightness Level", "Brightness_Value", -10, 10, function(v)
    if Config.Brightness_Enabled then Lighting.Brightness = v end
end)

-- 5. Setting
local Tab_Set = CreateTab("Setting")
AddToggle(Tab_Set, "Move Menu Button", "Menu_Move")
AddButton(Tab_Set, "Rename LocalPlayer (Client)", function()
    if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
        LocalPlayer.Character.Humanoid.DisplayName = "Admin"
    end
end)
AddSlider(Tab_Set, "Player Skin Size", "Player_Skin_Size", -50, 50, function(v)
    local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid")
    if hum then
        local scale = 1 + (v/100)
        local des = hum:GetAppliedDescription()
        if des then
            des.HeightScale = scale
            des.WidthScale = scale
            des.DepthScale = scale
            hum:ApplyDescription(des)
        end
    end
end)
AddButton(Tab_Set, "Toggle Skeleton Color (White/Red)", function()
    if Config.Color_ESP_Skeleton == Color3.new(1,1,1) then
        Config.Color_ESP_Skeleton = Color3.new(1,0,0)
    else
        Config.Color_ESP_Skeleton = Color3.new(1,1,1)
    end
end)

SwitchTab("Aimbot")
MainFrame.Visible = true

-- ■ Logic Core ■

UserInputService.JumpRequest:Connect(function()
    if Config.InfJump_Enabled then
        local char = LocalPlayer.Character
        if char then
            local hum = char:FindFirstChild("Humanoid")
            if hum then
                hum:ChangeState(Enum.HumanoidStateType.Jumping)
            end
        end
    end
end)

RunService.Stepped:Connect(function()
    if Config.Noclip_Enabled then
        local char = LocalPlayer.Character
        if char then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide then
                    part.CanCollide = false
                end
            end
        end
    end
end)

local function IsEnemy(player)
    if not Config.TeamCheck then return true end
    if player.Team == nil or LocalPlayer.Team == nil then return true end
    return player.Team ~= LocalPlayer.Team
end

local function GetClosestTarget()
    local closest = nil
    local shortest = Config.AimFOV_Enabled and Config.AimFOV_Radius or math.huge
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("Head") and IsEnemy(p) then
            local head = p.Character.Head
            local hum = p.Character:FindFirstChild("Humanoid")
            
            -- [NEW] Wall Check Logic
            if Config.WallCheck and not IsVisible(head, p.Character) then
                -- 見えない場合はスキップ
            elseif hum and hum.Health > 0 then
                local pos, onScreen = Camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                    if dist < shortest then
                        shortest = dist
                        closest = head
                    end
                end
            end
        end
    end
    return closest
end

-- ESP Object Management
local function CreateESPObj(plr)
    local esp = {
        Box = DrawingApi.new("Square"),
        Tracer = DrawingApi.new("Line"), 
        Skeleton = {},
        HeadTag = Instance.new("BillboardGui"),
        FeetTag = Instance.new("BillboardGui")
    }
    
    esp.Box.Visible = false
    esp.Box.Color = Config.Color_ESP_Box
    esp.Box.Thickness = 1.5
    esp.Box.Filled = false
    
    esp.Tracer.Visible = false
    esp.Tracer.Color = Config.Color_ESP_Line
    esp.Tracer.Thickness = 1
    
    for i=1, 15 do
        local l = DrawingApi.new("Line")
        l.Visible = false
        l.Thickness = 1
        l.Color = Config.Color_ESP_Skeleton
        table.insert(esp.Skeleton, l)
    end
    
    esp.HeadTag.Size = UDim2.new(0, 200, 0, 50)
    esp.HeadTag.StudsOffset = Vector3.new(0, 3, 0)
    esp.HeadTag.AlwaysOnTop = true
    esp.HeadTag.Parent = ScreenGui
    
    local nameLabel = Instance.new("TextLabel", esp.HeadTag)
    nameLabel.Size = UDim2.new(1,0,0,20)
    nameLabel.Position = UDim2.new(0,0,0,30)
    nameLabel.BackgroundTransparency = 1
    nameLabel.TextColor3 = Color3.new(1,1,1)
    nameLabel.TextStrokeTransparency = 0.5
    nameLabel.Font = Enum.Font.SourceSansBold
    nameLabel.TextSize = 14
    nameLabel.Text = ""
    esp.NameLabel = nameLabel
    
    local icon = Instance.new("ImageLabel", esp.HeadTag)
    icon.Size = UDim2.new(0, 30, 0, 30)
    icon.Position = UDim2.new(0.5, -15, 0, 0)
    icon.BackgroundTransparency = 1
    esp.FaceIcon = icon
    
    esp.FeetTag.Size = UDim2.new(0, 200, 0, 50)
    esp.FeetTag.StudsOffset = Vector3.new(0, -3.5, 0)
    esp.FeetTag.AlwaysOnTop = true
    esp.FeetTag.Parent = ScreenGui
    
    local infoLabel = Instance.new("TextLabel", esp.FeetTag)
    infoLabel.Size = UDim2.new(1,0,0,40)
    infoLabel.BackgroundTransparency = 1
    infoLabel.TextColor3 = Color3.fromRGB(0, 255, 100)
    infoLabel.TextStrokeTransparency = 0.5
    infoLabel.Font = Enum.Font.SourceSansBold
    infoLabel.TextSize = 13
    infoLabel.Text = ""
    esp.InfoLabel = infoLabel
    
    task.spawn(function()
        esp.FaceIcon.Image = Players:GetUserThumbnailAsync(plr.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size48x48)
        Instance.new("UICorner", esp.FaceIcon).CornerRadius = UDim.new(1,0)
    end)
    
    State.ESP_Storage[plr] = esp
end

local function RemoveESPObj(plr)
    if State.ESP_Storage[plr] then
        local esp = State.ESP_Storage[plr]
        esp.Box:Remove()
        esp.Tracer:Remove() 
        for _, l in pairs(esp.Skeleton) do l:Remove() end
        esp.HeadTag:Destroy()
        esp.FeetTag:Destroy()
        State.ESP_Storage[plr] = nil
    end
end

for _, p in pairs(Players:GetPlayers()) do if p ~= LocalPlayer then CreateESPObj(p) end end
Players.PlayerAdded:Connect(function(p) if p ~= LocalPlayer then CreateESPObj(p) end end)
Players.PlayerRemoving:Connect(RemoveESPObj)

-- ■ MAIN LOOP ■
RunService.RenderStepped:Connect(function()
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    
    -- 0. Crosshair Update
    if Config.Crosshair_Enabled then
        local size = Config.Crosshair_Size
        CrosshairH.From = center - Vector2.new(size, 0)
        CrosshairH.To = center + Vector2.new(size, 0)
        CrosshairV.From = center - Vector2.new(0, size)
        CrosshairV.To = center + Vector2.new(0, size)
        CrosshairH.Visible = true
        CrosshairV.Visible = true
    else
        CrosshairH.Visible = false
        CrosshairV.Visible = false
    end

    -- 1. FOV Circle
    FOVCircle.Position = center
    FOVCircle.Radius = Config.AimFOV_Radius
    FOVCircle.Visible = Config.AimFOV_Enabled
    
    -- 2. Aimbot Logic
    local shouldAim = false
    if Config.Aimbot_Enabled then
        if Config.TouchTrigger then
            shouldAim = State.Aiming
        else
            shouldAim = true
        end
    end
    
    if shouldAim then
        local target = GetClosestTarget()
        if target then
            local tPos = target.Position
            local currentCF = Camera.CFrame
            local desiredCF = CFrame.new(currentCF.Position, tPos)
            local smoothness = Config.Aimbot_Strength / 100
            if smoothness >= 1 then
                Camera.CFrame = desiredCF
            else
                Camera.CFrame = currentCF:Lerp(desiredCF, smoothness * 0.5)
            end
        end
    end
    
    -- 3. Player Stats Logic
    local char = LocalPlayer.Character
    if char then
        local hum = char:FindFirstChild("Humanoid")
        if hum then
            if Config.WalkSpeed_Enabled then 
                hum.WalkSpeed = Config.WalkSpeed_Value 
            end
            if Config.JumpPower_Enabled then 
                hum.UseJumpPower = true 
                hum.JumpPower = Config.JumpPower_Value 
            end
            if Config.FOV_Enabled then Camera.FieldOfView = Config.FOV_Value end
            if Config.Brightness_Enabled then Lighting.Brightness = Config.Brightness_Value end
        end
    end
    
    -- 4. ESP Update
    for plr, esp in pairs(State.ESP_Storage) do
        local char = plr.Character
        local hum = char and char:FindFirstChild("Humanoid")
        local root = char and char:FindFirstChild("HumanoidRootPart")
        local head = char and char:FindFirstChild("Head")
        local valid = char and hum and root and head and hum.Health > 0 and IsEnemy(plr)
        
        if valid then
            local pos, onScreen = Camera:WorldToViewportPoint(root.Position)
            local dist = (Camera.CFrame.Position - root.Position).Magnitude
            
            -- Box ESP
            if Config.ESP_Box and onScreen then
                local size = Vector3.new(2, 3, 0) * (2000 / math.max(dist, 1))
                local top, topVis = Camera:WorldToViewportPoint(root.Position + Vector3.new(0,3,0))
                local bot, botVis = Camera:WorldToViewportPoint(root.Position - Vector3.new(0,3,0))
                
                if topVis and botVis then
                    local height = math.abs(top.Y - bot.Y)
                    local width = height * 0.6
                    esp.Box.Size = Vector2.new(width, height)
                    esp.Box.Position = Vector2.new(pos.X - width/2, pos.Y - height/2)
                    esp.Box.Visible = true
                    esp.Box.Color = Config.Color_ESP_Box
                else esp.Box.Visible = false end
            else esp.Box.Visible = false end
            
            -- Tracer (Line) ESP
            if Config.ESP_Line and onScreen then
                esp.Tracer.From = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y) -- Bottom Center
                esp.Tracer.To = Vector2.new(pos.X, pos.Y)
                esp.Tracer.Visible = true
            else
                esp.Tracer.Visible = false
            end
            
            -- Skeleton ESP
            if Config.ESP_Skeleton and onScreen then
                local joints = {
                    {"Head", "UpperTorso"}, {"UpperTorso", "LowerTorso"},
                    {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"}, {"LeftLowerArm", "LeftHand"},
                    {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"}, {"RightLowerArm", "RightHand"},
                    {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"}, {"LeftLowerLeg", "LeftFoot"},
                    {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}, {"RightLowerLeg", "RightFoot"}
                }
                if hum.RigType == Enum.HumanoidRigType.R6 then
                     joints = {{"Head", "Torso"}, {"Torso", "Left Arm"}, {"Torso", "Right Arm"}, {"Torso", "Left Leg"}, {"Torso", "Right Leg"}}
                end
                
                for i, j in ipairs(joints) do
                    local p1 = char:FindFirstChild(j[1])
                    local p2 = char:FindFirstChild(j[2])
                    local line = esp.Skeleton[i]
                    if line then
                        if p1 and p2 then
                            local v1, vis1 = Camera:WorldToViewportPoint(p1.Position)
                            local v2, vis2 = Camera:WorldToViewportPoint(p2.Position)
                            if vis1 and vis2 then
                                line.From = Vector2.new(v1.X, v1.Y)
                                line.To = Vector2.new(v2.X, v2.Y)
                                line.Visible = true
                                line.Color = Config.Color_ESP_Skeleton
                            else line.Visible = false end
                        else line.Visible = false end
                    end
                end
            else for _, l in pairs(esp.Skeleton) do l.Visible = false end end
            
            -- Tags
            if onScreen then
                esp.HeadTag.Adornee = head; esp.HeadTag.Enabled = true
                esp.FeetTag.Adornee = root; esp.FeetTag.Enabled = true
                esp.NameLabel.Visible = Config.ESP_Name; esp.NameLabel.Text = plr.DisplayName or plr.Name
                esp.FaceIcon.Visible = Config.ESP_Face
                local infoText = ""
                if Config.ESP_HP then infoText = infoText .. "HP: " .. math.floor(hum.Health) .. "  " end
                if Config.ESP_Dist then infoText = infoText .. "Dst: " .. math.floor(dist) .. "m\n" end
                if Config.ESP_Item then
                    local tool = char:FindFirstChildOfClass("Tool")
                    if tool then infoText = infoText .. "[" .. tool.Name .. "]" end
                end
                esp.InfoLabel.Text = infoText
            else esp.HeadTag.Enabled = false; esp.FeetTag.Enabled = false end
        else
            esp.Box.Visible = false
            esp.Tracer.Visible = false
            for _, l in pairs(esp.Skeleton) do l.Visible = false end
            esp.HeadTag.Enabled = false
            esp.FeetTag.Enabled = false
        end
    end
    UpdateRadar()
end)
