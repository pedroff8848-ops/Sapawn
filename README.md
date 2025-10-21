-- LocalAdminSpawnUI_Preview.lua
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local getModelsFunc = ReplicatedStorage:WaitForChild("GetDevModels")
local requestPreview = ReplicatedStorage:WaitForChild("RequestPreview")
local spawnEvent = ReplicatedStorage:WaitForChild("AdminSpawnRequest")
local undoEvent = ReplicatedStorage:WaitForChild("AdminUndoRequest")
local previewsRoot = ReplicatedStorage:WaitForChild("ModelPreviews")

-- GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AdminSpawnUI_Stylish"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0,380,0,380)
frame.Position = UDim2.new(0,8,0,80)
frame.BackgroundColor3 = Color3.fromRGB(18,18,20)
frame.BorderSizePixel = 0
frame.Parent = screenGui

local title = Instance.new("TextLabel", frame)
title.Size = UDim2.new(1, -16, 0, 36)
title.Position = UDim2.new(0,8,0,8)
title.BackgroundTransparency = 1
title.Text = "Admin Spawn — Preview"
title.TextColor3 = Color3.fromRGB(220,220,220)
title.Font = Enum.Font.GothamBold
title.TextSize = 18
title.TextXAlignment = Enum.TextXAlignment.Left

-- Left: list
local listFrame = Instance.new("Frame", frame)
listFrame.Size = UDim2.new(0,170,1, -60)
listFrame.Position = UDim2.new(0,8,0,52)
listFrame.BackgroundColor3 = Color3.fromRGB(24,24,24)
listFrame.BorderSizePixel = 0

local scroll = Instance.new("ScrollingFrame", listFrame)
scroll.Size = UDim2.new(1, -12, 1, -12)
scroll.Position = UDim2.new(0,6,0,6)
scroll.BackgroundTransparency = 1
scroll.CanvasSize = UDim2.new(0,0,0,0)
scroll.ScrollBarThickness = 6

local uiList = Instance.new("UIListLayout", scroll)
uiList.Padding = UDim.new(0,6)
uiList.SortOrder = Enum.SortOrder.LayoutOrder

-- Right: preview area
local previewFrame = Instance.new("Frame", frame)
previewFrame.Size = UDim2.new(1, -196, 1, -60)
previewFrame.Position = UDim2.new(0,184,0,52)
previewFrame.BackgroundColor3 = Color3.fromRGB(30,30,30)
previewFrame.BorderSizePixel = 0

local previewTitle = Instance.new("TextLabel", previewFrame)
previewTitle.Size = UDim2.new(1, -12, 0, 24)
previewTitle.Position = UDim2.new(0,6,0,6)
previewTitle.BackgroundTransparency = 1
previewTitle.Text = "Preview"
previewTitle.TextColor3 = Color3.fromRGB(200,200,200)
previewTitle.Font = Enum.Font.GothamBold
previewTitle.TextSize = 14
previewTitle.TextXAlignment = Enum.TextXAlignment.Left

local viewport = Instance.new("ViewportFrame", previewFrame)
viewport.Size = UDim2.new(1, -12, 1, -64)
viewport.Position = UDim2.new(0,6,0,36)
viewport.BackgroundColor3 = Color3.fromRGB(18,18,18)
viewport.BorderSizePixel = 0

-- Bottom buttons
local spawnBtn = Instance.new("TextButton", frame)
spawnBtn.Size = UDim2.new(0,160,0,40)
spawnBtn.Position = UDim2.new(0,8,1, -44)
spawnBtn.Text = "Spawn Selected"
spawnBtn.Font = Enum.Font.Gotham
spawnBtn.TextSize = 15
spawnBtn.BackgroundColor3 = Color3.fromRGB(75,120,255)
spawnBtn.TextColor3 = Color3.fromRGB(255,255,255)
spawnBtn.BorderSizePixel = 0

local undoBtn = Instance.new("TextButton", frame)
undoBtn.Size = UDim2.new(0,160,0,40)
undoBtn.Position = UDim2.new(0,184,1, -44)
undoBtn.Text = "Undo (last)"
undoBtn.Font = Enum.Font.Gotham
undoBtn.TextSize = 15
undoBtn.BackgroundColor3 = Color3.fromRGB(180,60,60)
undoBtn.TextColor3 = Color3.fromRGB(255,255,255)
undoBtn.BorderSizePixel = 0

local selectedModel = nil
local modelButtons = {}

-- Helpers: limpa viewport
local function clearViewport()
    for _, c in ipairs(viewport:GetChildren()) do
        if not c:IsA("Camera") then c:Destroy() end
    end
end

-- Ajusta câmera do viewport para caber o modelo
local function fitCameraToModel(model)
    -- cria camera se não existir
    local cam = viewport:FindFirstChildOfClass("Camera") or Instance.new("Camera", viewport)
    viewport.CurrentCamera = cam

    -- calcula bounding box
    local minVec, maxVec
    for _, part in ipairs(model:GetDescendants()) do
        if part:IsA("BasePart") then
            local pos = part.Position
            local size = part.Size
            local pmin = pos - size/2
            local pmax = pos + size/2
            if not minVec then
                minVec = pmin
                maxVec = pmax
            else
                minVec = Vector3.new(math.min(minVec.X, pmin.X), math.min(minVec.Y, pmin.Y), math.min(minVec.Z, pmin.Z))
                maxVec = Vector3.new(math.max(maxVec.X, pmax.X), math.max(maxVec.Y, pmax.Y), math.max(maxVec.Z, pmax.Z))
            end
        end
    end

    if not minVec or not maxVec then
        -- fallback
        cam.CFrame = CFrame.new(Vector3.new(0,5,0) + Vector3.new(0,0,10), Vector3.new(0,0,0))
        return
    end

    local center = (minVec + maxVec) / 2
    local size = (maxVec - minVec)
    local radius = size.Magnitude / 2

    -- posiciona a camera um pouco afastada seguindo eixo Z negativo
    local distance = math.max(6, radius * 2.2)
    local camCFrame = CFrame.new(center + Vector3.new(0, 0, distance), center)
    cam.CFrame = camCFrame
end

-- Atualiza preview: espera clone em ReplicatedStorage.ModelPreviews/<userId>/<modelName>
local function updatePreviewFor(modelName)
    clearViewport()
    if not modelName then return end
    local userFolder = previewsRoot:FindFirstChild(tostring(player.UserId))
    if not userFolder then
        -- pedir ao servidor para criar
        requestPreview:FireServer(modelName)
        -- esperar pasta (até um timeout)
        local tries = 0
        repeat
            wait(0.15)
            userFolder = previewsRoot:FindFirstChild(tostring(player.UserId))
            tries = tries + 1
        until userFolder or tries > 12
        if not userFolder then return end
    end

    local previewModel = userFolder:FindFirstChild(modelName)
    if not previewModel then
        -- pedir ao servidor e tentar de novo
        requestPreview:FireServer(modelName)
        local tries = 0
        repeat
            wait(0.15)
            userFolder = previewsRoot:FindFirstChild(tostring(player.UserId))
            previewModel = userFolder and userFolder:FindFirstChild(modelName)
            tries = tries + 1
        until previewModel or tries > 12
        if not previewModel then return end
    end

    -- clone para dentro do viewport (não mover o original)
    local clone = previewModel:Clone()
    clone.Parent = viewport

    -- posiciona partes como anchored e non-collide (já feito no server, redundância)
    for _, d in ipairs(clone:GetDescendants()) do
        if d:IsA("BasePart") then
            d.Anchored = true
            d.CanCollide = false
        end
    end

    fitCameraToModel(clone)
end

-- Preenche lista chamando o RemoteFunction
local function refreshList()
    -- limpa UI antiga
    for _, child in ipairs(scroll:GetChildren()) do
        if child:IsA("TextButton") or child:IsA("Frame") then child:Destroy() end
    end
    modelButtons = {}

    local ok, result = pcall(function() return getModelsFunc:InvokeServer() end)
    if not ok or type(result) ~= "table" or #result == 0 then
        local lbl = Instance.new("TextLabel", scroll)
        lbl.Size = UDim2.new(1, -12, 0, 36)
        lbl.BackgroundTransparency = 1
        lbl.Text = "Sem modelos acessíveis ou sem permissão."
        lbl.TextColor3 = Color3.fromRGB(200,200,200)
        lbl.Font = Enum.Font.Gotham
        lbl.TextSize = 14
        scroll.CanvasSize = UDim2.new(0,0,0,40)
        return
    end

    for i, name in ipairs(result) do
        local item = Instance.new("Frame")
        item.Size = UDim2.new(1, -12, 0, 48)
        item.BackgroundColor3 = Color3.fromRGB(28,28,28)
        item.BorderSizePixel = 0
        item.Parent = scroll

        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(1, -12, 1, -12)
        btn.Position = UDim2.new(0,6,0,6)
        btn.BackgroundTransparency = 1
        btn.Text = ""
        btn.Parent = item

        -- small color tag / icon
        local tag = Instance.new("Frame", item)
        tag.Size = UDim2.new(0,6,1, -12)
        tag.Position = UDim2.new(0,6,0,6)
        tag.BackgroundColor3 = Color3.fromHSV((i * 0.12) % 1, 0.6, 0.9)
        tag.BorderSizePixel = 0

        local lbl = Instance.new("TextLabel", item)
        lbl.Size = UDim2.new(1, -24, 1, -12)
        lbl.Position = UDim2.new(0,16,0,6)
        lbl.BackgroundTransparency = 1
        lbl.Font = Enum.Font.Gotham
        lbl.TextSize = 14
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.TextColor3 = Color3.fromRGB(230,230,230)
        lbl.Text = name

        btn.MouseButton1Click:Connect(function()
            -- destaque
            for _, b in ipairs(modelButtons) do
                b.Frame.BackgroundColor3 = Color3.fromRGB(28,28,28)
            end
            item.BackgroundColor3 = Color3.fromRGB(44,44,60)
            selectedModel = name
            -- pedido de preview e update
            requestPreview:FireServer(name)
            updatePreviewFor(name)
        end)

        table.insert(modelButtons, {Frame = item, Name = name})
    end

    local cnt = #result
    scroll.CanvasSize = UDim2.new(0,0,0, cnt * 56 + 6)
end

-- spawn / undo
spawnBtn.MouseButton1Click:Connect(function()
    if not selectedModel then
        spawnBtn.Text = "Selecione um modelo primeiro"
        wait(1)
        spawnBtn.Text = "Spawn Selected"
        return
    end
    local pos = Vector3.new(0,5,0)
    if player.Character and player.Character.PrimaryPart then
        pos = player.Character.PrimaryPart.Position + (player.Character.PrimaryPart.CFrame.LookVector * 6) + Vector3.new(0,3,0)
    end
    spawnEvent:FireServer(selectedModel, pos)
end)

undoBtn.MouseButton1Click:Connect(function()
    undoEvent:FireServer()
end)

-- inicial
refreshList()

-- atalho R para refresh
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.R then refreshList() end
end)
