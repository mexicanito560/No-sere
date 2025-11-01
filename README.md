local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local player = Players.LocalPlayer

-- =========== CONFIG ===========
local API_URL = "https://brain-pro.vercel.app/all"
local UPDATE_INTERVAL = 2   -- segundos
local CARD_PADDING = 8
local CARD_HEIGHT = 90
local CARD_WIDTH = 360

local requestFunc = http_request or request or (syn and syn.request) or (http and http.request)
if not requestFunc then
    warn("[Finder] No http_request disponible.")
    return
end

local function inspectResponse(res)
    if type(res) == "table" then
        local status = res.StatusCode or res.status or res.code
        local body = res.Body or res.body
        return tonumber(status), tostring(body or "")
    elseif type(res) == "string" then
        return nil, res
    else
        return nil, tostring(res)
    end
end

local function fetchAPI()
    local ok, result = pcall(function()
        return requestFunc({
            Url = API_URL,
            Method = "GET",
            Headers = {
                ["User-Agent"] = "roblox-exploit-client/1.0",
                ["Accept"] = "application/json"
            }
        })
    end)

    if not ok then
        return false, "request_failed: " .. tostring(result)
    end

    local _, body = inspectResponse(result)
    if not body or body == "" then
        return false, "empty_body"
    end

    local success, data = pcall(function()
        return HttpService:JSONDecode(body)
    end)

    if not success then
        return false, "json_decode_failed"
    end

    local list = {}
    if type(data) == "table" then
        if #data > 0 then
            list = data
        else
            table.insert(list, data)
        end
    end

    return true, list
end

local function sortByTimestampDesc(list)
    table.sort(list, function(a, b)
        local ta = tostring(a and a.timestamp or "") 
        local tb = tostring(b and b.timestamp or "")
        if ta == "" and tb == "" then return false end
        if ta == "" then return false end
        if tb == "" then return true end
        return ta > tb -- mayor string = más reciente (ISO-8601)
    end)
end

-- === UI ===
local playerGui = player:WaitForChild("PlayerGui")
local UI_NAME = "DZ Finder"

local function removeOldUI()
    local old = playerGui:FindFirstChild(UI_NAME)
    if old then old:Destroy() end
end

local function createBaseUI()
    removeOldUI()

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = UI_NAME
    screenGui.ResetOnSpawn = false
    screenGui.Parent = playerGui

    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "Main"
    mainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
    mainFrame.Size = UDim2.new(0, 420, 0, 520)
    mainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
    mainFrame.BackgroundColor3 = Color3.fromRGB(28, 28, 30)
    mainFrame.BorderSizePixel = 0
    mainFrame.Parent = screenGui

    local title = Instance.new("TextLabel")
    title.Name = "Title"
    title.Size = UDim2.new(1, -20, 0, 36)
    title.Position = UDim2.new(0, 10, 0, 8)
    title.BackgroundTransparency = 1
    title.Text = "DZ Finder premiun"
    title.TextSize = 20
    title.Font = Enum.Font.GothamSemibold
    title.TextColor3 = Color3.fromRGB(235,235,235)
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = mainFrame

    local refreshBtn = Instance.new("TextButton")
    refreshBtn.Name = "RefreshBtn"
    refreshBtn.Size = UDim2.new(0, 90, 0, 28)
    refreshBtn.Position = UDim2.new(1, -100, 0, 8)
    refreshBtn.Text = "Actualizar"
    refreshBtn.Font = Enum.Font.Gotham
    refreshBtn.TextSize = 14
    refreshBtn.TextColor3 = Color3.fromRGB(255,255,255)
    refreshBtn.BackgroundColor3 = Color3.fromRGB(58,58,60)
    refreshBtn.BorderSizePixel = 0
    refreshBtn.Parent = mainFrame

    local statusLabel = Instance.new("TextLabel")
    statusLabel.Name = "Status"
    statusLabel.Size = UDim2.new(1, -20, 0, 20)
    statusLabel.Position = UDim2.new(0, 10, 0, 46)
    statusLabel.BackgroundTransparency = 1
    statusLabel.Text = "Última actualización: nunca"
    statusLabel.Font = Enum.Font.Gotham
    statusLabel.TextSize = 12
    statusLabel.TextColor3 = Color3.fromRGB(180,180,180)
    statusLabel.TextXAlignment = Enum.TextXAlignment.Left
    statusLabel.Parent = mainFrame

    local scroll = Instance.new("ScrollingFrame")
    scroll.Name = "Scroll"
    scroll.Size = UDim2.new(1, -20, 1, -80)
    scroll.Position = UDim2.new(0, 10, 0, 72)
    scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    scroll.ScrollBarThickness = 8
    scroll.BackgroundTransparency = 1
    scroll.Parent = mainFrame

    local layout = Instance.new("UIListLayout")
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, CARD_PADDING)
    layout.Parent = scroll

    layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        scroll.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + CARD_PADDING)
    end)

    local UIS = game:GetService("UserInputService")
    local dragging, dragInput, dragStart, startPos

    mainFrame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = mainFrame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    UIS.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)

    return screenGui, refreshBtn, statusLabel, scroll
end

local function createCard(parent, info, index)
    local card = Instance.new("Frame")
    card.Name = "dzcard"
    card.Size = UDim2.new(0, CARD_WIDTH, 0, CARD_HEIGHT)
    card.BackgroundColor3 = Color3.fromRGB(42, 42, 46)
    card.BorderSizePixel = 0
    card.LayoutOrder = index
    card.Parent = parent

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(0.6, -10, 0, 26)
    title.Position = UDim2.new(0, 8, 0, 8)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamSemibold
    title.TextSize = 16
    title.TextColor3 = Color3.fromRGB(240,240,240)
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Text = tostring(info.brainrotName or "Desconocido")
    title.Parent = card

    local rarity = Instance.new("TextLabel")
    rarity.Size = UDim2.new(0.35, -12, 0, 20)
    rarity.Position = UDim2.new(0, 8, 0, 36)
    rarity.BackgroundTransparency = 1
    rarity.Font = Enum.Font.Gotham
    rarity.TextSize = 14
    rarity.TextColor3 = Color3.fromRGB(200,200,255)
    rarity.TextXAlignment = Enum.TextXAlignment.Left
    rarity.Text = "Rareza: " .. tostring(info.rarity or "???")
    rarity.Parent = card

    local base = Instance.new("TextLabel")
    base.Size = UDim2.new(0.35, -12, 0, 20)
    base.Position = UDim2.new(0.6, -12, 0, 36)
    base.BackgroundTransparency = 1
    base.Font = Enum.Font.Gotham
    base.TextSize = 14
    base.TextColor3 = Color3.fromRGB(180,180,180)
    base.TextXAlignment = Enum.TextXAlignment.Right
    base.Text = tostring(info.base or "???")
    base.Parent = card

    local bottom = Instance.new("TextLabel")
    bottom.Size = UDim2.new(1, -16, 0, 16)
    bottom.Position = UDim2.new(0, 8, 0, 60)
    bottom.BackgroundTransparency = 1
    bottom.Font = Enum.Font.Gotham
    bottom.TextSize = 12
    bottom.TextColor3 = Color3.fromRGB(160,160,160)
    bottom.TextXAlignment = Enum.TextXAlignment.Left
    bottom.Text = string.format("PPS: %s | %s", tostring(info.value or ""), tostring(info.timestamp or ""))
    bottom.Parent = card

    local joinBtn = Instance.new("TextButton")
    joinBtn.Size = UDim2.new(0, 80, 0, 28)
    joinBtn.Position = UDim2.new(1, -88, 0, 8)
    joinBtn.BackgroundColor3 = Color3.fromRGB(64,160,255)
    joinBtn.BorderSizePixel = 0
    joinBtn.Font = Enum.Font.GothamSemibold
    joinBtn.TextSize = 14
    joinBtn.TextColor3 = Color3.fromRGB(255,255,255)
    joinBtn.Text = "Unirse"
    joinBtn.Parent = card

    joinBtn.MouseButton1Click:Connect(function()
        local placeId = game.PlaceId
        local jobId = tostring(info.jobId or "")
        if jobId == "" then return end
        local succ, err = pcall(function()
            TeleportService:TeleportToPlaceInstance(placeId, jobId, player)
        end)
        if not succ then
            warn("[WeroFinder] Teleport failed:", err)
        end
    end)

    -- ===== Estética avanzada (no cambia funcionalidad) =====
    -- Añadimos variaciones de color por rareza, avatares, badge, animaciones de entrada y hover profesional
    do
        local success, err = pcall(function()
            local TweenService = game:GetService("TweenService")

            -- Función auxiliar para mapear rareza a color (variaciones)
            local function rarityToColors(r)
                local s = (r or ""):lower()
                if s:find("legend") or s:find("legendary") then
                    return Color3.fromRGB(255, 200, 80), Color3.fromRGB(255, 170, 60)
                elseif s:find("epic") or s:find("purple") then
                    return Color3.fromRGB(190, 120, 255), Color3.fromRGB(150, 80, 240)
                elseif s:find("rare") then
                    return Color3.fromRGB(110, 180, 255), Color3.fromRGB(70, 140, 220)
                elseif s:find("uncommon") or s:find("green") then
                    return Color3.fromRGB(120, 220, 140), Color3.fromRGB(80, 180, 100)
                else
                    return Color3.fromRGB(200,200,200), Color3.fromRGB(160,160,160)
                end
            end

            local colorA, colorB = rarityToColors(tostring(info.rarity or ""))

            -- Rarity badge (lado derecho, pequeño y con gradiente)
            local badge = Instance.new("Frame")
            badge.Size = UDim2.new(0, 72, 0, 24)
            badge.Position = UDim2.new(1, -88, 0, 36)
            badge.AnchorPoint = Vector2.new(0, 0)
            badge.BackgroundTransparency = 0
            badge.BorderSizePixel = 0
            badge.Parent = card

            local badgeCorner = Instance.new("UICorner")
            badgeCorner.CornerRadius = UDim.new(0, 8)
            badgeCorner.Parent = badge

            local badgeGrad = Instance.new("UIGradient")
            badgeGrad.Color = ColorSequence.new{ColorSequenceKeypoint.new(0, colorA), ColorSequenceKeypoint.new(1, colorB)}
            badgeGrad.Rotation = 90
            badgeGrad.Parent = badge

            local badgeTxt = Instance.new("TextLabel")
            badgeTxt.Size = UDim2.new(1, -8, 1, 0)
            badgeTxt.Position = UDim2.new(0, 4, 0, 0)
            badgeTxt.BackgroundTransparency = 1
            badgeTxt.Font = Enum.Font.GothamSemibold
            badgeTxt.TextSize = 12
            badgeTxt.TextColor3 = Color3.fromRGB(30,30,30)
            badgeTxt.Text = tostring(info.rarity or "")
            badgeTxt.TextXAlignment = Enum.TextXAlignment.Center
            badgeTxt.Parent = badge

            -- Avatar (si viene info.avatarUrl, mostramos ImageLabel; no obligatorio)
            if info.avatarUrl and info.avatarUrl ~= "" then
                local avatar = Instance.new("ImageLabel")
                avatar.Size = UDim2.new(0, 56, 0, 56)
                avatar.Position = UDim2.new(0, 8, 0, 16)
                avatar.BackgroundTransparency = 1
                avatar.Image = info.avatarUrl
                avatar.Parent = card

                local aCorner = Instance.new("UICorner")
                aCorner.CornerRadius = UDim.new(0, 10)
                aCorner.Parent = avatar

                -- Shift title to the right to make space
                title.Position = UDim2.new(0, 72, 0, 8)
                title.Size = UDim2.new(0.55, -72, 0, 26)
            else
                -- si no hay avatar, dejamos icon decorativo
                local icon = Instance.new("TextLabel")
                icon.Size = UDim2.new(0, 28, 0, 28)
                icon.Position = UDim2.new(0, 8, 0, 8)
                icon.BackgroundTransparency = 1
                icon.Font = Enum.Font.GothamSemibold
                icon.TextSize = 16
                icon.Text = "🔎"
                icon.TextColor3 = colorA
                icon.Parent = card

                title.Position = UDim2.new(0, 40, 0, 8)
                title.Size = UDim2.new(0.6, -32, 0, 26)
            end

            -- Card corner, stroke y gradiente sutil
            local UICorner_card = Instance.new("UICorner")
            UICorner_card.CornerRadius = UDim.new(0, 12)
            UICorner_card.Parent = card

            local UIStroke_card = Instance.new("UIStroke")
            UIStroke_card.Thickness = 1
            UIStroke_card.Color = colorB
            UIStroke_card.Transparency = 0.8
            UIStroke_card.Parent = card

            local grad = Instance.new("UIGradient")
            grad.Color = ColorSequence.new{
                ColorSequenceKeypoint.new(0, Color3.fromRGB(46,46,50)),
                ColorSequenceKeypoint.new(1, Color3.fromRGB(38,38,42))
            }
            grad.Rotation = 270
            grad.Parent = card

            -- Join button styling: outline matching rarity
            local UICorner_btn = Instance.new("UICorner")
            UICorner_btn.CornerRadius = UDim.new(0, 8)
            UICorner_btn.Parent = joinBtn

            local UIStroke_btn = Instance.new("UIStroke")
            UIStroke_btn.Thickness = 1
            UIStroke_btn.Color = colorA
            UIStroke_btn.Transparency = 0.6
            UIStroke_btn.Parent = joinBtn

            -- Hover suave para el botón y ligero cambio de color
            local tweenInfo = TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
            joinBtn.MouseEnter:Connect(function()
                pcall(function()
                    TweenService:Create(joinBtn, tweenInfo, {BackgroundTransparency = 0.05, BackgroundColor3 = colorA}):Play()
                end)
            end)
            joinBtn.MouseLeave:Connect(function()
                pcall(function()
                    TweenService:Create(joinBtn, tweenInfo, {BackgroundTransparency = 0, BackgroundColor3 = Color3.fromRGB(64,160,255)}):Play()
                end)
            end)

            -- Hover en la tarjeta (elevar + sombra simulada con stroke)
            card.MouseEnter:Connect(function()
                pcall(function()
                    TweenService:Create(card, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.new(0, CARD_WIDTH, 0, CARD_HEIGHT + 4)}):Play()
                    UIStroke_card.Transparency = 0.4
                end)
            end)
            card.MouseLeave:Connect(function()
                pcall(function()
                    TweenService:Create(card, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.new(0, CARD_WIDTH, 0, CARD_HEIGHT)}):Play()
                    UIStroke_card.Transparency = 0.8
                end)
            end)

            -- Animación de entrada: fade + slide ligeramente desde la izquierda
            card.Position = UDim2.new(0, -40, 0, card.LayoutOrder * (CARD_HEIGHT + CARD_PADDING))
            card.BackgroundTransparency = 0.6
            TweenService:Create(card, TweenInfo.new(0.36, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {BackgroundTransparency = 0, Position = UDim2.new(0, 0, 0, (card.LayoutOrder - 1) * (CARD_HEIGHT + CARD_PADDING))}):Play()

        end)
        if not success then
            -- no rompemos nada si falla la estética
            -- warn("Estética avanzada falló:", err)
        end
    end

    return card
end

-- Rellenar la lista
local function populateCards(scroll, data)
    for _, child in ipairs(scroll:GetChildren()) do
        if child:IsA("Frame") and child.Name == "DZ Hub Deluxe" then
            child:Destroy()
        end
    end
    for i, info in ipairs(data) do
        createCard(scroll, info, i)
    end
end

-- === Main ===
local screenGui, refreshBtn, statusLabel, scrollFrame = createBaseUI()

-- ===== Aplicar mejoras estéticas globales (sin tocar lógica) =====
do
    local success, err = pcall(function()
        local TweenService = game:GetService("TweenService")
        local main = screenGui:WaitForChild("Main")
        -- 💫 Borde con brillo animado
        local glow = Instance.new("UIStroke")
        glow.Thickness = 2
        glow.Transparency = 0.7
        glow.Parent = main

        local glowGrad = Instance.new("UIGradient")
        glowGrad.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 230, 80)),  -- Amarillo claro
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 150)), -- Centro más brillante
            ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 230, 80))
        }
        glowGrad.Rotation = 0
        glowGrad.Parent = glow

        -- Animación infinita del brillo girando
        task.spawn(function()
            while task.wait(0.05) do
                glowGrad.Rotation = (glowGrad.Rotation + 3) % 360
            end
        end)

        -- Bordes redondeados y trazo sutil
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 12)
        corner.Parent = main

        local stroke = Instance.new("UIStroke")
        stroke.Thickness = 1
        stroke.Transparency = 0.9
        stroke.Parent = main

        -- Gradiente elegante de fondo
        local grad = Instance.new("UIGradient")
        grad.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0, Color3.fromRGB(30,30,32)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(22,22,24))
        }
        grad.Rotation = 90
        grad.Parent = main

        -- Escala general para que se vea más pulido en pantallas pequeñas
        local uiScale = Instance.new("UIScale")
        uiScale.Scale = 1
        uiScale.Parent = main

        -- Título: sombra sutil y kerning visual
        local titleLbl = main:FindFirstChild("Title")
        if titleLbl then
            titleLbl.TextSize = 20
            titleLbl.TextStrokeTransparency = 0.85
            titleLbl.TextTransparency = 0
        end

        -- Refresh button: esquinas, sombra y hover
        -- ⚡ Efecto de rayo de luz al pasar el mouse
        local lightGrad = Instance.new("UIGradient")
        lightGrad.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 255, 180)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 255)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 255, 180))
        }
        lightGrad.Rotation = 0
        lightGrad.Parent = refreshBtn

        refreshBtn.MouseEnter:Connect(function()
            pcall(function()
                for i = 1, 360, 15 do
                    lightGrad.Rotation = i
                    task.wait(0.02)
                end
            end)
        end)
        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 8)
        btnCorner.Parent = refreshBtn

        local btnStroke = Instance.new("UIStroke")
        btnStroke.Thickness = 1
        btnStroke.Transparency = 0.85
        btnStroke.Parent = refreshBtn

        local btnTweenIn = TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        refreshBtn.MouseEnter:Connect(function()
            pcall(function()
                TweenService:Create(refreshBtn, btnTweenIn, {BackgroundColor3 = Color3.fromRGB(78,140,255)}):Play()
            end)
        end)
        refreshBtn.MouseLeave:Connect(function()
            pcall(function()
                TweenService:Create(refreshBtn, btnTweenIn, {BackgroundColor3 = Color3.fromRGB(58,58,60)}):Play()
            end)
        end)

        -- Mejoras al Scroll (barra más sutil)
        local scroll = main:FindFirstChild("Scroll")
        if scroll then
            scroll.ScrollBarImageTransparency = 0.5
            scroll.ScrollBarThickness = 6
            scroll.BackgroundTransparency = 1
        end

        -- Pequeño pulido para status label
        local status = main:FindFirstChild("Status")
        if status then
            status.TextColor3 = Color3.fromRGB(150,150,150)
            status.TextSize = 12
        end
    end)
    if not success then
        -- no rompemos la lógica si falla la estética
        -- warn("Aplicar estética global falló:", err)
    end
end

local function updateOnce()
    statusLabel.Text = "Actualizando..."
    local ok, data = fetchAPI()
    if not ok then
        statusLabel.Text = "Error al obtener datos"
        return
    end

    if type(data) == "table" and #data > 0 then
        sortByTimestampDesc(data)
    end

    populateCards(scrollFrame, data)
    statusLabel.Text = "Última actualización: " .. os.date("%X")
end

refreshBtn.MouseButton1Click:Connect(updateOnce)

task.spawn(function()
    while true do
        local ok, err = pcall(updateOnce)
        if not ok then warn(err) end
        task.wait(UPDATE_INTERVAL)
    end
end)

print("[Finder] UI creada con auto-refresh cada " .. UPDATE_INTERVAL .. "s")
-- === Halo amarillo interior, suave y elegante ===
do
    local RunService = game:GetService("RunService")

    task.spawn(function()
        local player = game:GetService("Players").LocalPlayer
        local screenGui = player:WaitForChild("PlayerGui"):WaitForChild("DZ Finder")
        local main = screenGui:WaitForChild("Main")

        -- Crear halo
        local halo = Instance.new("Frame")
        halo.Name = "DZ_Halo"
        halo.Size = UDim2.new(0.97, 0, 0.97, 0) -- un poco más pequeño que el Main (ajusta el tamaño)
        halo.Position = UDim2.new(0.015, 0, 0.015, 0) -- centrado dentro del borde
        halo.AnchorPoint = Vector2.new(0, 0)
        halo.BackgroundTransparency = 1
        halo.ZIndex = 1000
        halo.Parent = main

        -- Contorno amarillo
        local stroke = Instance.new("UIStroke")
        stroke.Thickness = 3.5
        stroke.Color = Color3.fromRGB(255, 255, 0)
        stroke.Transparency = 0.25
        stroke.Parent = halo

        -- Bordes redondeados
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 12)
        corner.Parent = halo

        -- Variables para animación
        local rotation = 0
        local pulseSpeed = 2
        local timeElapsed = 0

        RunService.RenderStepped:Connect(function(dt)
            timeElapsed += dt
            rotation = (rotation + 80 * dt) % 360
            halo.Rotation = rotation

            -- Pulso suave
            local pulse = (math.sin(timeElapsed * pulseSpeed) + 1) / 2
            stroke.Transparency = 0.15 + (0.25 * (1 - pulse))
        end)
    end)
end
-- === Cuadrados flotantes amarillo suave (estilo futurista visible) ===
do
    local RunService = game:GetService("RunService")
    local player = game:GetService("Players").LocalPlayer

    task.spawn(function()
        local screenGui = player:WaitForChild("PlayerGui"):WaitForChild("DZ Finder")
        local main = screenGui:WaitForChild("Main")

        -- Carpeta para los cuadritos
        local particlesFolder = Instance.new("Folder")
        particlesFolder.Name = "DZ_SoftParticles"
        particlesFolder.Parent = main

        -- Crear cuadrados visibles arriba del fondo
        local totalSquares = 15
        local squares = {}

        for i = 1, totalSquares do
            local square = Instance.new("Frame")
            square.Size = UDim2.new(0, math.random(10, 25), 0, math.random(10, 25))
            square.Position = UDim2.new(math.random(), 0, math.random(), 0)
            square.AnchorPoint = Vector2.new(0.5, 0.5)
            square.BackgroundColor3 = Color3.fromRGB(255, 220, 80) -- amarillo bajo
            square.BackgroundTransparency = math.random(75, 90) / 100
            square.ZIndex = 5 -- 👈 esto hace que se vean arriba del fondo
            square.Parent = particlesFolder

            -- Bordes suaves (para que no sean cuadrados duros)
            local corner = Instance.new("UICorner")
            corner.CornerRadius = UDim.new(0, 4)
            corner.Parent = square

            squares[#squares + 1] = {
                frame = square,
                speedX = (math.random() - 0.5) * 0.04,
                speedY = (math.random() - 0.5) * 0.04,
            }
        end

        -- Movimiento + parpadeo suave
        RunService.RenderStepped:Connect(function(dt)
            for _, sq in ipairs(squares) do
                local frame = sq.frame
                local pos = frame.Position
                local newX = pos.X.Scale + sq.speedX
                local newY = pos.Y.Scale + sq.speedY

                -- Rebote suave dentro del Main
                if newX < 0 or newX > 1 then sq.speedX = -sq.speedX end
                if newY < 0 or newY > 1 then sq.speedY = -sq.speedY end

                frame.Position = UDim2.new(math.clamp(newX, 0, 1), 0, math.clamp(newY, 0, 1), 0)

                -- Efecto respirante de brillo (sutil)
                local t = tick() % 3
                frame.BackgroundTransparency = 0.75 + math.sin(t + newX * 5) * 0.1
            end
        end)
    end)
end
-- === Indicador FPS + Ping (esquina inferior derecha, transparente) ===
do
    local RunService = game:GetService("RunService")
    local Stats = game:GetService("Stats")
    local Players = game:GetService("Players")
    local player = Players.LocalPlayer

    task.spawn(function()
        local playerGui = player:WaitForChild("PlayerGui")

        -- Crear el ScreenGui separado (afuera de la UI principal)
        local statsGui = Instance.new("ScreenGui")
        statsGui.Name = "DZ_StatsHUD"
        statsGui.IgnoreGuiInset = true
        statsGui.ResetOnSpawn = false
        statsGui.Parent = playerGui

        -- Frame base (transparente)
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(0, 180, 0, 40)
        frame.Position = UDim2.new(1, -190, 1, -60)
        frame.BackgroundColor3 = Color3.fromRGB(30, 30, 32)
        frame.BackgroundTransparency = 0.65
        frame.BorderSizePixel = 0
        frame.Parent = statsGui

        -- Bordes suaves
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 8)
        corner.Parent = frame

        -- Texto
        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(1, -10, 1, 0)
        label.Position = UDim2.new(0, 5, 0, 0)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 14
        label.TextColor3 = Color3.fromRGB(255, 240, 140)
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.TextYAlignment = Enum.TextYAlignment.Center
        label.Text = "FPS: 0 │ Ping: 0 ms"
        label.Parent = frame

        -- Actualización
        local lastTime = tick()
        local frames = 0
        local fps = 0
        local ping = 0

        -- FPS counter
        RunService.RenderStepped:Connect(function()
            frames += 1
            local now = tick()
            if now - lastTime >= 1 then
                fps = frames
                frames = 0
                lastTime = now
            end
        end)

        -- Ping (desde Stats)
        local networkStats = Stats.Network.ServerStatsItem["Data Ping"]

        while task.wait(1) do
            if networkStats then
                ping = math.floor(networkStats:GetValue())
            end
            label.Text = string.format("🎮 FPS: %d  │  🌐 Ping: %d ms", fps, ping)
        end
    end)
end
-- Variables
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

-- Función para crear el ESP
local function CreateESP(Player)
    local Character = Player.Character
    if not Character then return end

    -- Crear el Highlight
    local Highlight = Instance.new("Highlight")
    Highlight.Name = "ESP_Highlight"
    Highlight.Adornee = Character
    Highlight.FillTransparency = 0.5 -- Hacer la skin visible
    Highlight.OutlineTransparency = 0
    Highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    Highlight.Parent = Character

    -- Crear el BillboardGui para el nombre
    local BillboardGui = Instance.new("BillboardGui")
    BillboardGui.Name = "ESP_Name"
    BillboardGui.Adornee = Character.Head
    BillboardGui.Size = UDim2.new(2, 0, 1, 0) -- Aumentar el tamaño
    BillboardGui.StudsOffset = Vector3.new(0, 2, 0)
    BillboardGui.AlwaysOnTop = true
    BillboardGui.Parent = Character.Head

    local TextLabel = Instance.new("TextLabel")
    TextLabel.Size = UDim2.new(1, 0, 1, 0)
    TextLabel.BackgroundTransparency = 0.5 -- Hacer el fondo transparente
    TextLabel.BackgroundColor3 = Color3.new(0, 0, 0) -- Fondo negro
    TextLabel.TextColor3 = Color3.new(1, 1, 1) -- Texto blanco
    TextLabel.TextScaled = true
    TextLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
    TextLabel.TextStrokeTransparency = 0
    TextLabel.Font = Enum.Font.SourceSansBold
    TextLabel.Text = Player.Name
    TextLabel.TextSize = 24 -- Aumentar el tamaño del texto
    TextLabel.Parent = BillboardGui

    -- Función para cambiar el color del Highlight
    local function UpdateRainbowColor()
        local hue = tick() % 1 -- Obtener un valor entre 0 y 1
        local rainbowColor = Color3.fromHSV(hue, 1, 1) -- Crear un color arcoíris
        Highlight.FillColor = rainbowColor
    end

    -- Actualizar el color cada segundo
    local lastUpdate = 0
    game:GetService("RunService").Heartbeat:Connect(function()
        if tick() - lastUpdate > 1 then
            UpdateRainbowColor()
            lastUpdate = tick()
        end
    end)
end

-- Función para eliminar el ESP
local function RemoveESP(Character)
    if Character then
        local Highlight = Character:FindFirstChild("ESP_Highlight")
        if Highlight then
            Highlight:Destroy()
        end
        local BillboardGui = Character.Head:FindFirstChild("ESP_Name")
        if BillboardGui then
            BillboardGui:Destroy()
        end
    end
end

-- Función para agregar ESP a un jugador
local function AddESP(Player)
    if Player ~= LocalPlayer then
        CreateESP(Player)
    end
end

-- Función para actualizar el ESP de todos los jugadores
local function UpdateAllESPs()
    for _, Player in pairs(Players:GetPlayers()) do
        if Player ~= LocalPlayer then
            local Character = Player.Character
            if Character then
                RemoveESP(Character) -- Eliminar el ESP anterior
                CreateESP(Player) -- Crear un nuevo ESP
            end
        end
    end
end

-- Agregar ESP a todos los jugadores existentes al inicio
UpdateAllESPs()

-- Agregar ESP cuando un nuevo jugador se une
Players.PlayerAdded:Connect(function(Player)
    Player.CharacterAdded:Connect(function(Character)
        wait(1) -- Esperar a que el personaje se cargue
        AddESP(Player)
    end)
end)

-- Eliminar ESP cuando un jugador se va
Players.PlayerRemoving:Connect(function(Player)
    RemoveESP(Player.Character)
end)

-- Actualizar el ESP cuando el personaje del jugador se une
local function onCharacterAdded(character)
    wait(1) -- Esperar a que el personaje se cargue
    local player = Players:GetPlayerFromCharacter(character)
    if player and player ~= LocalPlayer then
        CreateESP(player)
    end
end

-- Conectar el evento CharacterAdded para los jugadores existentes
for _, player in pairs(Players:GetPlayers()) do
    if player.Character then
        onCharacterAdded(player.Character)
    end
    player.CharacterAdded:Connect(onCharacterAdded)
end

-- Actualizar el ESP cada segundo
game:GetService("RunService").Heartbeat:Connect(function()
    UpdateAllESPs()
end)
-- 💛 DZ AutoFix Automático 💛
do
    local CoreGui = game:GetService("CoreGui")
    local TweenService = game:GetService("TweenService")

    local mainUIName = "DZ_MainUI" -- ⚠️ cambia esto por el nombre real de tu Frame principal
    local espName = "DZ_ESPSystem" -- si usas ESP pon su nombre, si no, déjalo así

    -- ✨ Función de notificación pro
    local function Notify(text, color)
        local n = Instance.new("TextLabel")
        n.Parent = CoreGui
        n.Size = UDim2.new(0, 300, 0, 30)
        n.Position = UDim2.new(0.5, -150, 1, -80)
        n.BackgroundTransparency = 0.2
        n.BackgroundColor3 = color or Color3.fromRGB(255, 255, 150)
        n.TextColor3 = Color3.fromRGB(20, 20, 20)
        n.Font = Enum.Font.GothamBold
        n.TextSize = 16
        n.Text = text
        local corner2 = Instance.new("UICorner", n)
        TweenService:Create(n, TweenInfo.new(0.4, Enum.EasingStyle.Quad), {Position = UDim2.new(0.5, -150, 1, -120)}):Play()
        task.wait(2.5)
        TweenService:Create(n, TweenInfo.new(0.4), {Transparency = 1, Position = UDim2.new(0.5, -150, 1, -60)}):Play()
        task.wait(0.5)
        n:Destroy()
    end

    -- 🧩 Función de reparación automática
    local function AutoRepair()
        local ui = CoreGui:FindFirstChild(mainUIName)
        local esp = CoreGui:FindFirstChild(espName)
        local needFix = false

        if not ui then
            warn("[DZ AutoFix] UI principal perdida.")
            needFix = true
        end
        if espName and not esp then
            warn("[DZ AutoFix] Sistema ESP no detectado.")
            needFix = true
        end

        if needFix then
            Notify("⚙️ DZ AutoFix: Reparando sistema...", Color3.fromRGB(255, 230, 90))
            local success, err = pcall(function()
                -- Aquí podrías volver a cargar tu hub si está online:
                -- loadstring(game:HttpGet("https://raw.githubusercontent.com/TUUSUARIO/TU_REPO/main/TU_HUB.lua"))()
                -- o simplemente re-crear los objetos:
                task.wait(1)
                if not ui then
                    local newUI = Instance.new("Frame", CoreGui)
                    newUI.Name = mainUIName
                    newUI.Size = UDim2.new(0, 500, 0, 300)
                    newUI.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
                end
                if espName and not esp then
                    local newESP = Instance.new("Folder", CoreGui)
                    newESP.Name = espName
                end
            end)

            if success then
                Notify("✅ DZ AutoFix: Sistema restaurado correctamente", Color3.fromRGB(150, 255, 150))
            else
                Notify("❌ Error durante reparación: " .. tostring(err), Color3.fromRGB(255, 150, 150))
            end
        end
    end

    -- 🔁 Escaneo automático cada 5 segundos
    task.spawn(function()
        while task.wait(5) do
            AutoRepair()
        end
    end)
end