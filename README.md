-- CoinClicker Hub
-- FUNC FIX: input adaptativo + loop leve; firesignal opcional, não obrigatório.

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local SoundService = game:GetService("SoundService")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local isMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

local oldSoundFolder = SoundService:FindFirstChild("CoinClickerHubSounds")
if oldSoundFolder then
    oldSoundFolder:Destroy()
end

local SoundFolder = Instance.new("Folder")
SoundFolder.Name = "CoinClickerHubSounds"
SoundFolder.Parent = SoundService

local function makeUiSound(name, soundId, volume, speed)
    local sound = Instance.new("Sound")
    sound.Name = name
    sound.SoundId = soundId
    sound.Volume = volume
    sound.PlaybackSpeed = speed
    sound.Parent = SoundFolder
    return sound
end

local HoverSound = makeUiSound(
    "Hover",
    "rbxassetid://7218169592",
    0.09,
    1.12
)

local ClickSound = makeUiSound(
    "Click",
    "rbxassetid://6976441148",
    0.17,
    0.96
)

local function playUiSound(sound)
    if not sound then
        return
    end

    sound:Stop()
    sound.TimePosition = 0
    sound:Play()
end

local State = {
    Running = true,
    AutoItems = false,
    AutoBuff = false,
    AutoWrinkler = false,
    AutoGolden = false,
    AutoBlackMarket = false,
    CompactView = false,

    LastItems = 0,
    LastBuff = 0,
    LastWrinkler = 0,
    LastGolden = 0,
    LastBlackMarket = 0,
    LastCompact = 0,

    ItemInterval = 1.20,
    BuffInterval = 1.40,
    WrinklerInterval = 0.85,
    GoldenInterval = 1.00,
    BlackMarketInterval = 0.45,
}

local Safety = {
    Paused = false,
    LastCheck = 0,
    CheckInterval = 0.45,
    TriggerText = nil,
    CleanChecks = 0,
}

local SeenWrinklers = setmetatable({}, { __mode = "k" })
local SeenGoldens = setmetatable({}, { __mode = "k" })

local function activeAutomationNames()
    local names = {}

    if State.AutoItems then table.insert(names, "Items") end
    if State.AutoBuff then table.insert(names, "Buff") end
    if State.AutoWrinkler then table.insert(names, "Wrinkler") end
    if State.AutoGolden then table.insert(names, "Golden") end
    if State.AutoBlackMarket then table.insert(names, "BlackMarket") end

    return table.concat(names, ", ")
end

local function warningObjectVisible(obj)
    if not obj
        or not (obj:IsA("TextLabel") or obj:IsA("TextButton"))
        or not obj.Visible
        or obj.TextTransparency >= 0.98
    then
        return false
    end

    local parent = obj.Parent

    while parent and parent ~= playerGui do
        if parent:IsA("GuiObject") and not parent.Visible then
            return false
        end

        if parent:IsA("LayerCollector") and not parent.Enabled then
            return false
        end

        parent = parent.Parent
    end

    local camera = workspace.CurrentCamera
    local viewport = camera and camera.ViewportSize

    if viewport then
        local pos = obj.AbsolutePosition
        local size = obj.AbsoluteSize

        if size.X <= 2
            or size.Y <= 2
            or pos.X + size.X < 0
            or pos.Y + size.Y < 0
            or pos.X > viewport.X
            or pos.Y > viewport.Y
        then
            return false
        end
    end

    return true
end

local function detectTooPerfectWarning()
    local gui = playerGui:FindFirstChild("CoinClickerGui")

    if not gui then
        return false, nil
    end

    for _, obj in ipairs(gui:GetDescendants()) do
        if warningObjectVisible(obj) then
            local raw = tostring(obj.Text or "")
            local txt = string.lower(raw)

            if string.find(txt, "too perfect", 1, true) then
                return true, raw
            end
        end
    end

    return false, nil
end

local function safetyCheck(now)
    if now - Safety.LastCheck < Safety.CheckInterval then
        return Safety.Paused
    end

    Safety.LastCheck = now

    local found, warningText = detectTooPerfectWarning()

    if found then
        Safety.CleanChecks = 0

        if not Safety.Paused then
            warn(
                "[CoinClicker Hub] Automacoes pausadas enquanto 'Too Perfect' estiver visivel. Ativas: "
                .. activeAutomationNames()
            )
        end

        Safety.Paused = true
        Safety.TriggerText = warningText
    else
        Safety.CleanChecks += 1

        if Safety.Paused and Safety.CleanChecks >= 2 then
            Safety.Paused = false
            Safety.TriggerText = nil
        end
    end

    return Safety.Paused
end


--==================================================
-- FUNÇÕES DE SUPORTE & PARSER [MELHORIA 5]
--==================================================
local NumberSuffixes = {
    [""] = 1,
    k = 1e3,
    m = 1e6,
    b = 1e9,
    t = 1e12,
    qa = 1e15,
    qi = 1e18,
    sx = 1e21,
    sp = 1e24,
    oc = 1e27,
    no = 1e30,
    dc = 1e33,
}

local function parseNumber(text)
    if not text or text == "" then
        return 0
    end

    local lower = string.lower(tostring(text))
    lower = string.gsub(lower, ",", ".")

    local value, suffix = string.match(lower, "(%d+%.?%d*)%s*(%a*)")

    if not value then
        return 0
    end

    local number = tonumber(value) or 0
    local multiplier = NumberSuffixes[suffix or ""] or 1

    return number * multiplier
end

local function getGui()
    return playerGui:FindFirstChild("CoinClickerGui")
end

local function visible(obj)
    if not obj or not obj:IsA("GuiObject") or not obj.Visible then return false end
    local p = obj.Parent
    while p and p ~= playerGui do
        if p:IsA("GuiObject") and not p.Visible then return false end
        p = p.Parent
    end
    return true
end

local MenuGate = {
    open = false,
    lastCheck = 0,
}

local function isCoinClickerMenuOpen(now)
    now = now or os.clock()

    if now - MenuGate.lastCheck < 0.15 then
        return MenuGate.open
    end

    MenuGate.lastCheck = now

    local gui = getGui()
    if not gui then
        MenuGate.open = false
        return false
    end

    if gui:IsA("ScreenGui") and not gui.Enabled then
        MenuGate.open = false
        return false
    end

    -- CoinClickerClient registra esse proxy no UiScreenManager.
    -- Ele é o indicador mais confiável de que o menu está realmente aberto.
    local proxy = gui:FindFirstChild("ScreenProxy", true)

    if proxy and proxy:IsA("GuiObject") then
        MenuGate.open = proxy.Visible
        return MenuGate.open
    end

    -- Fallback para versões em que o proxy muda de nome.
    local main = gui:FindFirstChild("Main", true)
    local left = gui:FindFirstChild("LeftColumn", true)
    local middle = gui:FindFirstChild("MiddleColumn", true)
    local right = gui:FindFirstChild("RightColumn", true)

    local anyColumn =
        (left and visible(left))
        or (middle and visible(middle))
        or (right and visible(right))

    MenuGate.open = main ~= nil and visible(main) and anyColumn ~= nil
    return MenuGate.open
end


--==================================================
-- INPUT COMPAT - POTASSIUM / XENO / FALLBACK
-- Ordem:
-- 1) firesignal, quando existir;
-- 2) VirtualInputManager via serviço;
-- 3) VirtualInputManager via Instance.new (compat com executores que expõem assim);
-- 4) VirtualUser;
-- 5) GuiButton:Activate().
--==================================================
local VirtualUser = nil

pcall(function()
    VirtualUser = game:GetService("VirtualUser")
end)

local function getVirtualInputManager()
    local vim = nil
    local created = false

    pcall(function()
        vim = game:GetService("VirtualInputManager")
    end)

    if not vim then
        pcall(function()
            vim = Instance.new("VirtualInputManager")
            created = true
        end)
    end

    return vim, created
end

local function setHiddenAncestorsVisible(button)
    local changed = {}
    local parent = button

    while parent and parent ~= playerGui do
        if parent:IsA("GuiObject") and not parent.Visible then
            table.insert(changed, {
                object = parent,
                visible = parent.Visible,
            })
            parent.Visible = true
        end

        parent = parent.Parent
    end

    return changed
end

local function restoreHiddenAncestors(changed)
    for i = #changed, 1, -1 do
        local item = changed[i]

        if item.object and item.object.Parent then
            item.object.Visible = item.visible
        end
    end
end

local function getButtonCenter(button)
    local size = button.AbsoluteSize
    local pos = button.AbsolutePosition

    if size.X <= 2 or size.Y <= 2 then
        return nil
    end

    return math.floor(pos.X + size.X * 0.5),
        math.floor(pos.Y + size.Y * 0.5)
end

local function clickWithVirtualInput(button)
    local x, y = getButtonCenter(button)
    if not x then
        return false
    end

    local vim, created = getVirtualInputManager()
    if not vim then
        return false
    end

    local ok = pcall(function()
        vim:SendMouseMoveEvent(x, y, game)
        task.wait(0.01)
        vim:SendMouseButtonEvent(x, y, 0, true, game, 0)
        task.wait(0.025)
        vim:SendMouseButtonEvent(x, y, 0, false, game, 0)
    end)

    if created and vim then
        pcall(function()
            vim:Destroy()
        end)
    end

    return ok
end

local function clickWithVirtualUser(button)
    if not VirtualUser then
        return false
    end

    local x, y = getButtonCenter(button)
    if not x then
        return false
    end

    local point = Vector2.new(x, y)
    local camera = workspace.CurrentCamera
    local cameraCFrame = camera and camera.CFrame or CFrame.new()

    local ok = pcall(function()
        VirtualUser:CaptureController()
        VirtualUser:ClickButton1(point, cameraCFrame)
    end)

    if ok then
        return true
    end

    return pcall(function()
        VirtualUser:CaptureController()
        VirtualUser:Button1Down(point, cameraCFrame)
        task.wait(0.025)
        VirtualUser:Button1Up(point, cameraCFrame)
    end)
end

local function compatActivate(button, allowHidden)
    if not button
        or not button:IsA("GuiButton")
        or not button.Parent
    then
        return false
    end

    local changed = {}

    if allowHidden then
        changed = setHiddenAncestorsVisible(button)
        task.wait(0.02)
    elseif not visible(button) then
        return false
    end

    -- Mantém compatibilidade com executores completos sem tornar isso obrigatório.
    if type(firesignal) == "function" then
        local ok = pcall(function()
            firesignal(button.Activated)
        end)

        restoreHiddenAncestors(changed)

        if ok then
            return true
        end
    end

    -- Xeno e similares podem expor VirtualInputManager de formas diferentes.
    if clickWithVirtualInput(button) then
        restoreHiddenAncestors(changed)
        return true
    end

    if clickWithVirtualUser(button) then
        restoreHiddenAncestors(changed)
        return true
    end

    -- Último fallback, totalmente padrão.
    local oldActive = button.Active

    if not oldActive then
        button.Active = true
    end

    local ok = pcall(function()
        button:Activate()
    end)

    if button and button.Parent then
        button.Active = oldActive
    end

    restoreHiddenAncestors(changed)

    return ok
end

local function press(button)
    if not button
        or not button:IsA("GuiButton")
        or not visible(button)
    then
        return false
    end

    return compatActivate(button, false)
end

local function findFrame(name)
    local gui = getGui()
    if not gui then return nil end
    return gui:FindFirstChild(name, true)
end

local FrameCache = {}

local function getFrame(name)
    local cached = FrameCache[name]

    if cached and cached.Parent then
        return cached
    end

    cached = findFrame(name)
    FrameCache[name] = cached
    return cached
end

local function allButtons(root)
    local result = {}
    if not root then return result end
    if root:IsA("GuiButton") and visible(root) then
        table.insert(result, root)
    end
    for _, obj in ipairs(root:GetDescendants()) do
        if obj:IsA("GuiButton") and visible(obj) then
            table.insert(result, obj)
        end
    end
    return result
end

local function textOf(obj)
    local parts = {}
    if obj:IsA("TextButton") and obj.Text ~= "" then
        table.insert(parts, obj.Text)
    end
    for _, d in ipairs(obj:GetDescendants()) do
        if (d:IsA("TextLabel") or d:IsA("TextButton")) and d.Text ~= "" then
            table.insert(parts, d.Text)
        end
    end
    return string.lower(table.concat(parts, " "))
end

local function normalizedButtonText(button)
    local txt = string.lower(textOf(button))
    txt = string.gsub(txt, "^%s+", "")
    txt = string.gsub(txt, "%s+$", "")
    txt = string.gsub(txt, "%s+", " ")
    return txt
end

local function findNativeFortunesButton()
    local gui = getGui()
    if not gui then
        return nil
    end

    local best = nil
    local bestArea = math.huge

    for _, obj in ipairs(gui:GetDescendants()) do
        if obj:IsA("GuiButton") then
            local txt = normalizedButtonText(obj)
            local name = string.lower(obj.Name)

            if txt == "fortunes"
                or name == "fortunes"
                or string.find(name, "fortune", 1, true)
            then
                local area = obj.AbsoluteSize.X * obj.AbsoluteSize.Y

                if area > 0 and area < bestArea then
                    best = obj
                    bestArea = area
                end
            end
        end
    end

    return best
end

local function activateNativeButton(button)
    if not button or not button:IsA("GuiButton") then
        return false
    end

    -- Fortunes pode estar dentro de uma coluna escondida no Compact View.
    return compatActivate(button, true)
end

local function openNativeFortunes()
    return activateNativeButton(findNativeFortunesButton())
end

local function greenStroke(obj)
    local function green(c)
        return c.G > c.R * 1.08 and c.G > c.B * 1.02 and c.G > 0.4
    end
    for _, d in ipairs(obj:GetDescendants()) do
        if d:IsA("UIStroke") and d.Enabled and green(d.Color) then return true end
    end
    local p = obj.Parent
    if p then
        for _, d in ipairs(p:GetChildren()) do
            if d:IsA("UIStroke") and d.Enabled and green(d.Color) then return true end
        end
    end
    return false
end

local function getCoinBalance()
    local left = getFrame("LeftColumn")
    if not left then
        return nil
    end

    local best = nil

    for _, obj in ipairs(left:GetDescendants()) do
        if (obj:IsA("TextLabel") or obj:IsA("TextButton")) and visible(obj) then
            local txt = string.lower(obj.Text or "")

            if string.find(txt, "coin", 1, true)
                and not string.find(txt, "per second", 1, true)
                and not string.find(txt, "/s", 1, true)
            then
                local value = parseNumber(txt)

                if value > 0 and (best == nil or value > best) then
                    best = value
                end
            end
        end
    end

    return best
end

local function getLargestPlainNumber(root, skipPerSecond, skipLeft)
    local best = 0

    if not root then
        return best
    end

    for _, obj in ipairs(root:GetDescendants()) do
        if obj:IsA("TextLabel") or obj:IsA("TextButton") then
            local txt = string.lower(obj.Text or "")

            if txt ~= ""
                and (not skipPerSecond or not string.find(txt, "/s", 1, true))
                and (not skipLeft or not string.find(txt, "left", 1, true))
            then
                local value = parseNumber(txt)

                if value > best then
                    best = value
                end
            end
        end
    end

    return best
end

local function findCloseButton(root)
    if not root then
        return nil
    end

    local best = nil
    local bestArea = math.huge

    for _, button in ipairs(allButtons(root)) do
        local txt = string.lower(textOf(button))
        local name = string.lower(button.Name)

        local isClose =
            txt == "x"
            or txt == "×"
            or txt == "close"
            or txt == "fechar"
            or name == "close"
            or string.find(name, "close", 1, true) ~= nil

        if isClose then
            local area = button.AbsoluteSize.X * button.AbsoluteSize.Y

            if area < bestArea then
                best = button
                bestArea = area
            end
        end
    end

    return best
end

--==================================================
--==================================================
--==================================================
-- COMPRA INTELIGENTE [MELHORIA 5]
--==================================================
local function buyBestGenerator()
    local generators = getFrame("Generators")
    if not generators then
        return false
    end

    local balance = getCoinBalance()
    if not balance then
        return false
    end

    local candidates = {}

    for _, button in ipairs(allButtons(generators)) do
        local size = button.AbsoluteSize

        if size.X >= 120 and size.Y >= 32 then
            local txt = textOf(button)

            if not string.find(txt, "robux", 1, true) then
                local price = parseNumber(txt)
                local affordable =
                    price > 0
                    and price <= balance
                    and greenStroke(button)

                if affordable then
                    table.insert(candidates, {
                        button = button,
                        price = price,
                        posY = button.AbsolutePosition.Y,
                    })
                end
            end
        end
    end

    -- Compra somente quando existe algo realmente disponível.
    table.sort(candidates, function(a, b)
        if a.price ~= b.price then
            return a.price > b.price
        end

        return a.posY > b.posY
    end)

    local choice = candidates[1]

    if choice then
        return press(choice.button)
    end

    return false
end

--==================================================
-- BUFF / UPGRADES
--==================================================
local LastUpgradeButton = nil
local LastUpgradeButtonAt = 0

local function buyAvailableUpgrade()
    local upgrades = getFrame("Upgrades")
    if not upgrades then
        return false
    end

    local candidates = allButtons(upgrades)

    table.sort(candidates, function(a, b)
        return a.AbsolutePosition.X < b.AbsolutePosition.X
    end)

    local now = os.clock()

    for _, button in ipairs(candidates) do
        if greenStroke(button) then
            if button == LastUpgradeButton and now - LastUpgradeButtonAt < 2.0 then
                continue
            end

            if press(button) then
                LastUpgradeButton = button
                LastUpgradeButtonAt = now
                return true
            end
        end
    end

    return false
end

--==================================================
-- WRINKLERS / GOLDENS
--==================================================
local function popWrinklers()
    local wrinklers = getFrame("Wrinklers")
    if not wrinklers then
        return 0
    end

    for _, button in ipairs(allButtons(wrinklers)) do
        local size = button.AbsoluteSize

        if size.X >= 20
            and size.Y >= 20
            and not SeenWrinklers[button]
        then
            if press(button) then
                SeenWrinklers[button] = true
                return 1
            end
        end
    end

    return 0
end

local function clickGoldens()
    local goldens = getFrame("Goldens")
    if not goldens then
        return 0
    end

    for _, button in ipairs(allButtons(goldens)) do
        if not SeenGoldens[button] then
            if press(button) then
                SeenGoldens[button] = true
                return 1
            end
        end
    end

    return 0
end

--==================================================
-- BLACK MARKET - CICLO CONTROLADO
-- Abre uma vez, compra somente o que cabe no saldo e fecha.
--==================================================

local BlackMarketCache = {
    modal = nil,
    openButton = nil,
}

local BlackMarketFlow = {
    phase = "idle",
    nextAction = 0,
    openedByAuto = false,
}

local function normalizeText(value)
    local s = string.lower(tostring(value or ""))
    s = string.gsub(s, "%s+", " ")
    return s
end

local function objectText(obj)
    if not obj then
        return ""
    end

    local parts = {}

    if (obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox"))
        and obj.Text ~= ""
    then
        table.insert(parts, obj.Text)
    end

    for _, d in ipairs(obj:GetDescendants()) do
        if (d:IsA("TextLabel") or d:IsA("TextButton") or d:IsA("TextBox"))
            and d.Text ~= ""
        then
            table.insert(parts, d.Text)
        end
    end

    return normalizeText(table.concat(parts, " "))
end

local function findBlackMarketModal()
    local cached = BlackMarketCache.modal

    if cached
        and cached.Parent
        and cached:IsA("GuiObject")
        and visible(cached)
    then
        return cached
    end

    BlackMarketCache.modal = nil

    local gui = getGui()
    if not gui then
        return nil
    end

    for _, obj in ipairs(gui:GetDescendants()) do
        if (obj:IsA("TextLabel") or obj:IsA("TextButton"))
            and visible(obj)
        then
            local txt = normalizeText(obj.Text)

            if txt == "black market"
                or string.find(txt, "black market", 1, true)
            then
                local p = obj.Parent

                while p and p ~= gui do
                    if p:IsA("GuiObject") then
                        local size = p.AbsoluteSize
                        local allText = objectText(p)

                        if size.X >= 280
                            and size.Y >= 220
                            and (
                                string.find(allText, "closing soon", 1, true)
                                or string.find(allText, "gone when the timer", 1, true)
                                or string.find(allText, " left", 1, true)
                            )
                        then
                            BlackMarketCache.modal = p
                            return p
                        end
                    end

                    p = p.Parent
                end
            end
        end
    end

    return nil
end

local function findBlackMarketOpenButton()
    local cached = BlackMarketCache.openButton

    if cached
        and cached.Parent
        and cached:IsA("GuiButton")
        and visible(cached)
    then
        return cached
    end

    BlackMarketCache.openButton = nil

    local gui = getGui()
    if not gui then
        return nil
    end

    for _, obj in ipairs(gui:GetDescendants()) do
        if obj:IsA("GuiButton") and visible(obj) then
            local txt = normalizeText(textOf(obj))

            if txt == "open" or string.find(txt, " open ", 1, true) then
                local p = obj.Parent

                for _ = 1, 5 do
                    if not p or p == gui then
                        break
                    end

                    local parentText = objectText(p)

                    if string.find(parentText, "black market", 1, true)
                        or string.find(parentText, "rare goods for sale", 1, true)
                    then
                        BlackMarketCache.openButton = obj
                        return obj
                    end

                    p = p.Parent
                end
            end
        end
    end

    return nil
end

local function findBlackMarketCard(label, modal)
    local p = label.Parent

    while p and p ~= modal do
        if p:IsA("GuiObject") then
            local size = p.AbsoluteSize
            local txt = objectText(p)

            if size.X >= 180
                and size.Y >= 42
                and size.Y <= 150
                and string.find(txt, "/s", 1, true)
                and string.find(txt, "left", 1, true)
            then
                return p
            end
        end

        p = p.Parent
    end

    return nil
end

local function findLargestButton(root)
    if not root then
        return nil
    end

    if root:IsA("GuiButton") and visible(root) then
        return root
    end

    local best = nil
    local bestArea = 0

    for _, button in ipairs(allButtons(root)) do
        local area = button.AbsoluteSize.X * button.AbsoluteSize.Y

        if area > bestArea then
            best = button
            bestArea = area
        end
    end

    return best
end

local function getBlackMarketEntries(modal)
    local entries = {}
    local seenCards = {}

    for _, obj in ipairs(modal:GetDescendants()) do
        if obj:IsA("TextLabel") and visible(obj) then
            local txt = normalizeText(obj.Text)

            if string.find(txt, "left", 1, true) then
                local card = findBlackMarketCard(obj, modal)

                if card and not seenCards[card] then
                    seenCards[card] = true

                    local button = findLargestButton(card)
                    local price = getLargestPlainNumber(card, true, true)

                    local leftCount = tonumber(string.match(txt, "(%d+)%s*left")) or 1

                    if button and price > 0 and leftCount > 0 then
                        table.insert(entries, {
                            card = card,
                            button = button,
                            price = price,
                            left = leftCount,
                            y = card.AbsolutePosition.Y,
                        })
                    end
                end
            end
        end
    end

    table.sort(entries, function(a, b)
        if a.price ~= b.price then
            return a.price > b.price
        end

        return a.y < b.y
    end)

    return entries
end

local function resetBlackMarketFlow(cooldown)
    BlackMarketFlow.phase = "idle"
    BlackMarketFlow.openedByAuto = false
    BlackMarketFlow.nextAction = os.clock() + (cooldown or 0)
    BlackMarketCache.modal = nil
end

local function stepBlackMarket(now)
    if now < BlackMarketFlow.nextAction then
        return
    end

    local modal = findBlackMarketModal()

    if not modal then
        local openButton = findBlackMarketOpenButton()

        if openButton then
            if press(openButton) then
                BlackMarketFlow.phase = "opening"
                BlackMarketFlow.openedByAuto = true
                BlackMarketFlow.nextAction = now + 0.65
                BlackMarketCache.modal = nil
            end
        else
            BlackMarketFlow.nextAction = now + 2.0
        end

        return
    end

    if BlackMarketFlow.phase == "opening" then
        BlackMarketFlow.phase = "buying"
        BlackMarketFlow.nextAction = now + 0.25
        return
    end

    local balance = getCoinBalance()
    local entries = getBlackMarketEntries(modal)

    if balance then
        for _, entry in ipairs(entries) do
            if entry.price <= balance then
                if press(entry.button) then
                    BlackMarketFlow.phase = "buying"
                    BlackMarketFlow.nextAction = now + 0.75
                    return
                end
            end
        end
    end

    -- Nada comprável: fecha o modal que o automático abriu e espera.
    if BlackMarketFlow.openedByAuto then
        local closeButton = findCloseButton(modal)

        if closeButton then
            press(closeButton)
        end
    end

    resetBlackMarketFlow(8.0)
end

--==================================================
-- COMPACT VIEW
--==================================================
local saved = setmetatable({}, {__mode = "k"})

local function remember(obj)
    if not obj or not obj:IsA("GuiObject") or saved[obj] then return end
    saved[obj] = {
        Visible = obj.Visible,
        Size = obj.Size,
        Position = obj.Position,
        AnchorPoint = obj.AnchorPoint,
        BackgroundTransparency = obj.BackgroundTransparency,
        Active = obj.Active,
        CompactScale = obj:FindFirstChild("CoinClickerCompactScale"),
    }
end

local function restore(obj)
    local old = saved[obj]
    if not old or not obj or not obj.Parent then return end
    obj.Visible = old.Visible
    obj.Size = old.Size
    obj.Position = old.Position
    obj.AnchorPoint = old.AnchorPoint
    obj.BackgroundTransparency = old.BackgroundTransparency
    obj.Active = old.Active
end

local function setCompactScale(obj, scaleValue)
    if not obj or not obj:IsA("GuiObject") then
        return
    end

    local scaler = obj:FindFirstChild("CoinClickerCompactScale")

    if not scaler then
        scaler = Instance.new("UIScale")
        scaler.Name = "CoinClickerCompactScale"
        scaler.Parent = obj
    end

    scaler.Scale = scaleValue
end

local function clearCompactScale(obj)
    if not obj then
        return
    end

    local scaler = obj:FindFirstChild("CoinClickerCompactScale")
    if scaler then
        scaler:Destroy()
    end
end

local CompactFortunesButton = nil

local function restoreCompactLayout()
    for obj, _ in pairs(saved) do
        if obj and obj.Parent and obj:IsA("GuiObject") then
            restore(obj)
        end
    end

    table.clear(saved)
end

local function applyCompact()
    local gui = getGui()

    if not gui then
        if CompactFortunesButton then
            CompactFortunesButton.Visible = false
        end
        return
    end

    local main = gui:FindFirstChild("Main", true)
    local left = gui:FindFirstChild("LeftColumn", true)
    local middle = gui:FindFirstChild("MiddleColumn", true)
    local right = gui:FindFirstChild("RightColumn", true)

    -- Esses botões ficam direto no Main, fora das colunas.
    -- Por isso mover Left/Right não alterava a posição deles.
    local changelogBtn = main and main:FindFirstChild("ChangelogBtn")
    local closeBtn = main and main:FindFirstChild("CloseBtn")

    for _, obj in ipairs({
        main,
        left,
        middle,
        right,
        changelogBtn,
        closeBtn,
    }) do
        if obj and obj:IsA("GuiObject") then
            remember(obj)
        end
    end

    if State.CompactView then
        -- Reserva espaço no topo para os botões nativos do Roblox/CoinClicker.
        -- Antes as colunas começavam em Y=6 e ficavam por cima dos controles do topo.
        local topSafeOffset = isMobile and 64 or 54
        local bottomSafeMargin = 10
        local topButtonY = isMobile and 72 or 62

        if main then
            main.BackgroundTransparency = 1
            main.Active = false
        end

        -- Move SOMENTE os controles próprios do CoinClicker para baixo,
        -- deixando espaço para o HUD original do jogo no canto superior.
        if changelogBtn and changelogBtn:IsA("GuiObject") then
            changelogBtn.Position = UDim2.new(1, -88, 0, topButtonY)
        end

        if closeBtn and closeBtn:IsA("GuiObject") then
            closeBtn.Position = UDim2.new(1, -48, 0, topButtonY)
        end

        if left then
            left.Visible = true
            left.AnchorPoint = Vector2.new(0, 0)
            left.Position = UDim2.fromOffset(6, topSafeOffset)
            left.Size = UDim2.new(
                0,
                isMobile and 235 or 220,
                1,
                -(topSafeOffset + bottomSafeMargin)
            )
            setCompactScale(left, isMobile and 0.90 or 0.86)
        end

        -- A coluna central continua escondida no Compact View.
        if middle then
            middle.Visible = false
            middle.Active = false
        end

        if right then
            right.Visible = true
            right.AnchorPoint = Vector2.new(1, 0)
            right.Position = UDim2.new(1, -6, 0, topSafeOffset)
            right.Size = UDim2.new(
                0,
                isMobile and 260 or 245,
                1,
                -(topSafeOffset + bottomSafeMargin)
            )
            setCompactScale(right, isMobile and 0.90 or 0.86)
        end

        if CompactFortunesButton and left and isCoinClickerMenuOpen() then
            local viewport = workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize
            local buttonHeight = isMobile and 38 or 36
            local availableWidth = math.max(130, math.floor(left.AbsoluteSize.X - 12))
            local buttonWidth = math.min(isMobile and 210 or 196, availableWidth)

            -- Fica ABAIXO da coluna esquerda, naquele espaço vazio da tela.
            local x = math.floor(
                left.AbsolutePosition.X
                + (left.AbsoluteSize.X - buttonWidth) / 2
            )

            local desiredY = math.floor(left.AbsolutePosition.Y + left.AbsoluteSize.Y + 6)

            if viewport then
                desiredY = math.min(desiredY, viewport.Y - buttonHeight - 12)
            end

            CompactFortunesButton.Size = UDim2.fromOffset(buttonWidth, buttonHeight)
            CompactFortunesButton.Position = UDim2.fromOffset(x, desiredY)
            CompactFortunesButton.Visible = true
            CompactFortunesButton.Active = true
        elseif CompactFortunesButton then
            CompactFortunesButton.Visible = false
        end
    else
        clearCompactScale(left)
        clearCompactScale(right)
        restoreCompactLayout()

        if CompactFortunesButton then
            CompactFortunesButton.Visible = false
        end
    end
end

--==================================================
-- INTERFACE (HUB) - COMPACT DARK
--==================================================

local old = playerGui:FindFirstChild("CoinClickerHubCustom")
if old then
    old:Destroy()
end

local Gui = Instance.new("ScreenGui")
Gui.Name = "CoinClickerHubCustom"
Gui.ResetOnSpawn = false
Gui.IgnoreGuiInset = true
Gui.DisplayOrder = 1000
Gui.Parent = playerGui

CompactFortunesButton = Instance.new("TextButton")
CompactFortunesButton.Name = "CompactFortunesButton"
CompactFortunesButton.Size = UDim2.fromOffset(184, 36)
CompactFortunesButton.Position = UDim2.fromOffset(8, 8)
CompactFortunesButton.BackgroundColor3 = Color3.fromRGB(18, 19, 22)
CompactFortunesButton.BorderSizePixel = 0
CompactFortunesButton.Text = "Fortunes"
CompactFortunesButton.TextColor3 = Color3.fromRGB(220, 220, 224)
CompactFortunesButton.Font = Enum.Font.GothamMedium
CompactFortunesButton.TextSize = 13
CompactFortunesButton.AutoButtonColor = false
CompactFortunesButton.Visible = false
CompactFortunesButton.ZIndex = 20
CompactFortunesButton.Parent = Gui

local CompactFortunesCorner = Instance.new("UICorner")
CompactFortunesCorner.CornerRadius = UDim.new(0, 5)
CompactFortunesCorner.Parent = CompactFortunesButton

local CompactFortunesStroke = Instance.new("UIStroke")
CompactFortunesStroke.Thickness = 1
CompactFortunesStroke.Color = Color3.fromRGB(74, 76, 82)
CompactFortunesStroke.Parent = CompactFortunesButton

CompactFortunesButton.MouseEnter:Connect(function()
    CompactFortunesButton.BackgroundColor3 = Color3.fromRGB(27, 28, 31)
end)

CompactFortunesButton.MouseLeave:Connect(function()
    CompactFortunesButton.BackgroundColor3 = Color3.fromRGB(14, 15, 17)
end)

CompactFortunesButton.Activated:Connect(function()
    if not isCoinClickerMenuOpen() then
        return
    end

    openNativeFortunes()
end)


local function tween(obj, info, props)
    local ok, tw = pcall(function()
        return TweenService:Create(obj, info, props)
    end)

    if ok and tw then
        tw:Play()
        return tw
    end

    return nil
end

local Main = Instance.new("Frame")
Main.Size = isMobile
    and UDim2.fromOffset(320, 320)
    or UDim2.fromOffset(500, 232)
Main.Position = isMobile
    and UDim2.new(0.5, -160, 0.5, -160)
    or UDim2.new(0.5, -250, 0.5, -116)
Main.BackgroundColor3 = Color3.fromRGB(8, 9, 11)
Main.BorderSizePixel = 0
Main.Active = true
Main.Draggable = true
Main.Parent = Gui

local HubScale = Instance.new("UIScale")
HubScale.Scale = 0.94
HubScale.Parent = Main

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = Main

local MainStroke = Instance.new("UIStroke")
MainStroke.Thickness = 2
MainStroke.Color = Color3.fromRGB(94, 96, 104)
MainStroke.Transparency = 0.05
MainStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
MainStroke.LineJoinMode = Enum.LineJoinMode.Round
MainStroke.Parent = Main

local OuterStroke = Instance.new("UIStroke")
OuterStroke.Name = "OuterStroke"
OuterStroke.Thickness = 1
OuterStroke.Color = Color3.fromRGB(40, 41, 46)
OuterStroke.Transparency = 0.18
OuterStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
OuterStroke.LineJoinMode = Enum.LineJoinMode.Round
OuterStroke.Parent = Main

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 36)
Header.BackgroundColor3 = Color3.fromRGB(10, 11, 13)
Header.BorderSizePixel = 0
Header.Parent = Main

local HeaderCorner = Instance.new("UICorner")
HeaderCorner.CornerRadius = UDim.new(0, 10)
HeaderCorner.Parent = Header

local HeaderStroke = Instance.new("UIStroke")
HeaderStroke.Thickness = 1
HeaderStroke.Color = Color3.fromRGB(52, 54, 59)
HeaderStroke.Transparency = 0.30
HeaderStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
HeaderStroke.Parent = Header

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(0, isMobile and 112 or 126, 1, 0)
Title.Position = UDim2.fromOffset(10, 0)
Title.BackgroundTransparency = 1
Title.Text = "CoinClicker Hub"
Title.TextColor3 = Color3.fromRGB(190, 190, 195)
Title.Font = Enum.Font.GothamMedium
Title.TextSize = isMobile and 12 or 14
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Header

local CreditWrap = Instance.new("Frame")
CreditWrap.Name = "CreditWrap"
CreditWrap.Size = UDim2.fromOffset(isMobile and 84 or 96, 18)
CreditWrap.Position = UDim2.fromOffset(isMobile and 117 or 132, 9)
CreditWrap.BackgroundTransparency = 1
CreditWrap.Parent = Header

local DiscordLogo = Instance.new("ImageLabel")
DiscordLogo.Name = "DiscordLogo"
DiscordLogo.Size = UDim2.fromOffset(18, 18)
DiscordLogo.Position = UDim2.fromOffset(0, 0)
DiscordLogo.BackgroundTransparency = 1
DiscordLogo.ScaleType = Enum.ScaleType.Fit
DiscordLogo.Parent = CreditWrap

DiscordLogo.Image = "rbxthumb://type=Asset&id=126084249376054&w=150&h=150"

local CreditText = Instance.new("TextLabel")
CreditText.Name = "CreditText"
CreditText.Position = UDim2.fromOffset(23, 0)
CreditText.Size = UDim2.new(1, -23, 1, 0)
CreditText.BackgroundTransparency = 1
CreditText.Text = "by 00018y"
CreditText.TextColor3 = Color3.fromRGB(225, 225, 230)
CreditText.Font = Enum.Font.Gotham
CreditText.TextSize = isMobile and 9 or 10
CreditText.TextXAlignment = Enum.TextXAlignment.Left
CreditText.Parent = CreditWrap

local Activity = Instance.new("TextLabel")
Activity.Size = UDim2.fromOffset(70, 22)
Activity.Position = UDim2.new(1, -142, 0, 7)
Activity.BackgroundColor3 = Color3.fromRGB(36, 37, 40)
Activity.BorderSizePixel = 0
Activity.Text = "INATIVO"
Activity.TextColor3 = Color3.fromRGB(175, 176, 180)
Activity.Font = Enum.Font.GothamBold
Activity.TextSize = 9
Activity.Parent = Header

local ActivityCorner = Instance.new("UICorner")
ActivityCorner.CornerRadius = UDim.new(1, 0)
ActivityCorner.Parent = Activity

local Minimize = Instance.new("TextButton")
Minimize.Size = UDim2.fromOffset(30, 24)
Minimize.Position = UDim2.new(1, -66, 0, 6)
Minimize.BackgroundColor3 = Color3.fromRGB(20, 21, 24)
Minimize.BorderSizePixel = 0
Minimize.Text = "—"
Minimize.TextColor3 = Color3.fromRGB(170, 170, 175)
Minimize.Font = Enum.Font.GothamBold
Minimize.TextSize = 15
Minimize.Parent = Header

local MinCorner = Instance.new("UICorner")
MinCorner.CornerRadius = UDim.new(0, 4)
MinCorner.Parent = Minimize

local MinStroke = Instance.new("UIStroke")
MinStroke.Thickness = 1
MinStroke.Color = Color3.fromRGB(58, 60, 65)
MinStroke.Transparency = 0.24
MinStroke.Parent = Minimize

local Close = Instance.new("TextButton")
Close.Size = UDim2.fromOffset(30, 24)
Close.Position = UDim2.new(1, -34, 0, 6)
Close.BackgroundColor3 = Color3.fromRGB(20, 21, 24)
Close.BorderSizePixel = 0
Close.Text = "×"
Close.TextColor3 = Color3.fromRGB(200, 125, 135)
Close.Font = Enum.Font.GothamBold
Close.TextSize = 16
Close.Parent = Header

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 4)
CloseCorner.Parent = Close

local CloseStroke = Instance.new("UIStroke")
CloseStroke.Thickness = 1
CloseStroke.Color = Color3.fromRGB(75, 55, 59)
CloseStroke.Transparency = 0.20
CloseStroke.Parent = Close

local Body = Instance.new("Frame")
Body.Position = UDim2.fromOffset(4, 40)
Body.Size = UDim2.new(1, -8, 1, -44)
Body.BackgroundTransparency = 1
Body.Parent = Main

local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0, isMobile and 58 or 68, 1, 0)
Sidebar.BackgroundTransparency = 1
Sidebar.Parent = Body

local SidebarBasePosition = Sidebar.Position

local SideLayout = Instance.new("UIListLayout")
SideLayout.Padding = UDim.new(0, 4)
SideLayout.Parent = Sidebar

local Content = Instance.new("Frame")
Content.Position = UDim2.fromOffset(isMobile and 63 or 73, 0)
local ContentBasePosition = Content.Position
Content.Size = UDim2.new(1, -(isMobile and 63 or 73), 1, 0)
Content.BackgroundColor3 = Color3.fromRGB(11, 12, 14)
Content.BorderSizePixel = 0
Content.Parent = Body

local ContentCorner = Instance.new("UICorner")
ContentCorner.CornerRadius = UDim.new(0, 8)
ContentCorner.Parent = Content

local ContentStroke = Instance.new("UIStroke")
ContentStroke.Thickness = 1
ContentStroke.Color = Color3.fromRGB(43, 45, 49)
ContentStroke.Parent = Content

local function makePage()
    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.fromScale(1, 1)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = isMobile and 3 or 4
    page.CanvasSize = UDim2.fromOffset(0, 0)
    page.AutomaticCanvasSize = Enum.AutomaticSize.Y
    page.ScrollingDirection = Enum.ScrollingDirection.Y
    page.Parent = Content

    local padding = Instance.new("UIPadding")
    padding.PaddingTop = UDim.new(0, 5)
    padding.PaddingBottom = UDim.new(0, 5)
    padding.PaddingLeft = UDim.new(0, 5)
    padding.PaddingRight = UDim.new(0, 5)
    padding.Parent = page

    local grid = Instance.new("UIGridLayout")
    grid.CellPadding = UDim2.fromOffset(4, 4)
    grid.CellSize = isMobile
        and UDim2.new(1, -3, 0, 43)
        or UDim2.new(0.5, -2, 0, 53)
    grid.SortOrder = Enum.SortOrder.LayoutOrder
    grid.Parent = page

    return page
end

local FarmPage = makePage()
local ViewPage = makePage()
ViewPage.Visible = false

local navButtons = {}

local function setPage(page)
    FarmPage.Visible = page == FarmPage
    ViewPage.Visible = page == ViewPage

    -- Pequeno slide lateral, no estilo RoVibes.
    Content.Position = ContentBasePosition + UDim2.fromOffset(7, 0)
    tween(Content, TweenInfo.new(0.16, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
        Position = ContentBasePosition
    })

    for button, target in pairs(navButtons) do
        local selected = target == page

        tween(button, TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = selected
                and Color3.fromRGB(47, 48, 52)
                or Color3.fromRGB(15, 16, 18),
            TextColor3 = selected
                and Color3.fromRGB(238, 238, 241)
                or Color3.fromRGB(150, 151, 156)
        })

        local accent = button:FindFirstChild("ActiveAccent")
        if accent then
            tween(accent, TweenInfo.new(0.14, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
                Size = selected
                    and UDim2.new(0, 3, 0.62, 0)
                    or UDim2.new(0, 3, 0, 0),
                BackgroundTransparency = selected and 0.15 or 1
            })
        end
    end
end

local function navButton(textValue, page)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(1, 0, 0, isMobile and 31 or 28)
    button.BackgroundColor3 = Color3.fromRGB(15, 16, 18)
    button.BorderSizePixel = 0
    button.Text = textValue
    button.TextColor3 = Color3.fromRGB(150, 151, 156)
    button.Font = Enum.Font.GothamMedium
    button.TextSize = isMobile and 10 or 10
    button.Parent = Sidebar

    local Accent = Instance.new("Frame")
    Accent.Name = "ActiveAccent"
    Accent.AnchorPoint = Vector2.new(0, 0.5)
    Accent.Position = UDim2.new(0, 2, 0.5, 0)
    Accent.Size = UDim2.new(0, 3, 0, 0)
    Accent.BackgroundColor3 = Color3.fromRGB(208, 208, 214)
    Accent.BackgroundTransparency = 1
    Accent.BorderSizePixel = 0
    Accent.Parent = button

    local AccentCorner = Instance.new("UICorner")
    AccentCorner.CornerRadius = UDim.new(1, 0)
    AccentCorner.Parent = Accent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 4)
    corner.Parent = button

    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 1
    stroke.Color = Color3.fromRGB(40, 42, 46)
    stroke.Parent = button

    navButtons[button] = page

    button.MouseEnter:Connect(function()
        if button.BackgroundColor3 ~= Color3.fromRGB(47, 48, 52) then
            tween(button, TweenInfo.new(0.10, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                BackgroundColor3 = Color3.fromRGB(23, 24, 27)
            })
        end
    end)

    button.MouseLeave:Connect(function()
        local selected = navButtons[button] == (FarmPage.Visible and FarmPage or ViewPage)

        tween(button, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = selected
                and Color3.fromRGB(47, 48, 52)
                or Color3.fromRGB(15, 16, 18)
        })
    end)

    button.Activated:Connect(function()
        setPage(page)
    end)

    return button
end

navButton("Farm", FarmPage)
navButton("View", ViewPage)

local function makeToggle(parent, labelText, key, layoutOrder)
    local Card = Instance.new("Frame")
    Card.LayoutOrder = layoutOrder or 0
    Card.BackgroundColor3 = Color3.fromRGB(14, 15, 17)
    Card.BorderSizePixel = 0
    Card.Parent = parent

    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(0, 7)
    Corner.Parent = Card

    local Stroke = Instance.new("UIStroke")
    Stroke.Thickness = 1
    Stroke.Color = Color3.fromRGB(39, 41, 45)
    Stroke.Parent = Card

    Card.MouseEnter:Connect(function()
        tween(Card, TweenInfo.new(0.10, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = Color3.fromRGB(18, 19, 22)
        })
        tween(Stroke, TweenInfo.new(0.10, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Color = Color3.fromRGB(54, 56, 61)
        })
    end)

    Card.MouseLeave:Connect(function()
        tween(Card, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = Color3.fromRGB(14, 15, 17)
        })
        tween(Stroke, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Color = Color3.fromRGB(39, 41, 45)
        })
    end)

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, -64, 1, 0)
    Label.Position = UDim2.fromOffset(8, 0)
    Label.BackgroundTransparency = 1
    Label.Text = labelText
    Label.TextColor3 = Color3.fromRGB(205, 205, 210)
    Label.Font = Enum.Font.GothamMedium
    Label.TextSize = isMobile and 10 or 10
    Label.TextWrapped = true
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.Parent = Card

    local Toggle = Instance.new("TextButton")
    Toggle.Size = UDim2.fromOffset(isMobile and 44 or 48, 18)
    Toggle.Position = UDim2.new(1, -(isMobile and 52 or 56), 0.5, -9)
    Toggle.BackgroundColor3 = Color3.fromRGB(38, 39, 42)
    Toggle.BorderSizePixel = 0
    Toggle.Text = ""
    Toggle.AutoButtonColor = false
    Toggle.Parent = Card

    local ToggleCorner = Instance.new("UICorner")
    ToggleCorner.CornerRadius = UDim.new(1, 0)
    ToggleCorner.Parent = Toggle

    local ToggleStroke = Instance.new("UIStroke")
    ToggleStroke.Thickness = 1
    ToggleStroke.Color = Color3.fromRGB(59, 61, 66)
    ToggleStroke.Transparency = 0.24
    ToggleStroke.Parent = Toggle

    local Knob = Instance.new("Frame")
    Knob.Size = UDim2.fromOffset(14, 14)
    Knob.Position = UDim2.fromOffset(2, 2)
    Knob.BackgroundColor3 = Color3.fromRGB(210, 210, 214)
    Knob.BorderSizePixel = 0
    Knob.Parent = Toggle

    local KnobCorner = Instance.new("UICorner")
    KnobCorner.CornerRadius = UDim.new(1, 0)
    KnobCorner.Parent = Knob

    local function refresh(animated)
        local on = State[key]

        local toggleColor = on
            and Color3.fromRGB(68, 70, 74)
            or Color3.fromRGB(38, 39, 42)

        local knobPosition = on
            and UDim2.new(1, -16, 0, 2)
            or UDim2.fromOffset(2, 2)

        if animated then
            tween(Toggle, TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                BackgroundColor3 = toggleColor
            })

            tween(Knob, TweenInfo.new(0.14, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Position = knobPosition
            })

            tween(Knob, TweenInfo.new(0.10, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                Size = UDim2.fromOffset(15, 15)
            })

            task.delay(0.10, function()
                if Knob and Knob.Parent then
                    tween(Knob, TweenInfo.new(0.10, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                        Size = UDim2.fromOffset(14, 14)
                    })
                end
            end)
        else
            Toggle.BackgroundColor3 = toggleColor
            Knob.Position = knobPosition
        end
    end

    Toggle.Activated:Connect(function()
        State[key] = not State[key]

        if key == "CompactView" then
            pcall(applyCompact)
        end

        refresh(true)
    end)

    refresh(false)
end

makeToggle(FarmPage, "Auto Comprar Itens", "AutoItems", 1)
makeToggle(FarmPage, "Auto Buff / Upgrades", "AutoBuff", 2)
makeToggle(FarmPage, "Auto Wrinklers", "AutoWrinkler", 3)
makeToggle(FarmPage, "Auto Golden", "AutoGolden", 4)
makeToggle(FarmPage, "Auto Black Market", "AutoBlackMarket", 5)
makeToggle(ViewPage, "Compact View", "CompactView", 1)

setPage(FarmPage)

local SoundBound = setmetatable({}, { __mode = "k" })
local LastHoverAt = 0

local function bindHubSound(button)
    if not button:IsA("GuiButton") or SoundBound[button] then
        return
    end

    SoundBound[button] = true

    button.MouseEnter:Connect(function()
        local now = os.clock()

        if now - LastHoverAt >= 0.035 then
            LastHoverAt = now
            playUiSound(HoverSound)
        end
    end)

    button.Activated:Connect(function()
        playUiSound(ClickSound)
    end)
end

for _, obj in ipairs(Gui:GetDescendants()) do
    if obj:IsA("GuiButton") then
        bindHubSound(obj)
    end
end

Gui.DescendantAdded:Connect(function(obj)
    if obj:IsA("GuiButton") then
        task.defer(bindHubSound, obj)
    end
end)

local finalMainPosition = Main.Position
local finalTitlePosition = Title.Position

Main.Position = finalMainPosition + UDim2.fromOffset(0, 10)
Main.BackgroundTransparency = 1
Title.Position = finalTitlePosition + UDim2.fromOffset(-8, 0)
Title.TextTransparency = 1
Sidebar.Position = SidebarBasePosition + UDim2.fromOffset(-8, 0)
Content.Position = ContentBasePosition + UDim2.fromOffset(10, 0)

tween(HubScale, TweenInfo.new(0.22, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
    Scale = 1
})

tween(Main, TweenInfo.new(0.20, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
    Position = finalMainPosition,
    BackgroundTransparency = 0
})

tween(Title, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
    Position = finalTitlePosition,
    TextTransparency = 0
})

tween(Sidebar, TweenInfo.new(0.20, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
    Position = SidebarBasePosition
})

tween(Content, TweenInfo.new(0.22, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
    Position = ContentBasePosition
})

local lastMenuVisual = nil

local function setActivity(active)
    if Safety.Paused then
        Activity.Text = "PAUSADO"
        tween(Activity, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            TextColor3 = Color3.fromRGB(245, 176, 176),
            BackgroundColor3 = Color3.fromRGB(68, 34, 36)
        })
        tween(MainStroke, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Color = Color3.fromRGB(92, 48, 52)
        })
        return
    end

    if lastMenuVisual == active then
        return
    end

    lastMenuVisual = active

    if active then
        Activity.Text = "ATIVO"
        tween(Activity, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            TextColor3 = Color3.fromRGB(160, 230, 178),
            BackgroundColor3 = Color3.fromRGB(29, 54, 38)
        })
        tween(MainStroke, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Color = Color3.fromRGB(58, 78, 65)
        })
    else
        Activity.Text = "INATIVO"
        tween(Activity, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            TextColor3 = Color3.fromRGB(175, 176, 180),
            BackgroundColor3 = Color3.fromRGB(36, 37, 40)
        })
        tween(MainStroke, TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Color = Color3.fromRGB(48, 50, 54)
        })
    end
end

--==================================================
-- LOOP ÚNICO
-- Fora do menu CoinClicker: nenhuma automação executa.
--==================================================

local lastMenuOpen = nil
local compactReapplyAt = 0

local function clearCoinClickerUiCaches()
    table.clear(FrameCache)

    BlackMarketCache.modal = nil
    BlackMarketCache.openButton = nil

end

local function resetAutomationFlows()
    BlackMarketFlow.phase = "idle"
    BlackMarketFlow.openedByAuto = false
    BlackMarketFlow.nextAction = 0
end

local function handleMenuTransition(menuOpen, now)
    if lastMenuOpen == menuOpen then
        return
    end

    lastMenuOpen = menuOpen
    clearCoinClickerUiCaches()
    resetAutomationFlows()

    if not menuOpen then
        -- Muito importante: devolve a UI original antes do CoinClicker fechar.
        -- Isso evita botões nativos ficarem inativos na próxima abertura.
        clearCompactScale(getFrame("LeftColumn"))
        clearCompactScale(getFrame("RightColumn"))
        restoreCompactLayout()

        if CompactFortunesButton then
            CompactFortunesButton.Visible = false
        end

        compactReapplyAt = 0
    else
        -- Dá um instante para o CoinClicker reconstruir os controles.
        compactReapplyAt = now + 0.18
    end
end


--==================================================
-- L HUB - LOCAL OVERHEAD TAG
-- Só aparece no personagem do LocalPlayer.
--==================================================
local LHubTagName = "LHubLocalTag"
local LHubCharacterConnection = nil

local function clearLHubTag()
    local oldTag = playerGui:FindFirstChild(LHubTagName)
    if oldTag then
        oldTag:Destroy()
    end
end

local function lHubCorner(parent, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius)
    corner.Parent = parent
    return corner
end

local function lHubStroke(parent, color, thickness, transparency)
    local s = Instance.new("UIStroke")
    s.Color = color
    s.Thickness = thickness
    s.Transparency = transparency or 0
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Parent = parent
    return s
end

local function createLHubTag(character)
    clearLHubTag()

    local head = character:WaitForChild("Head", 10)
    if not head then
        return
    end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = LHubTagName
    billboard.Adornee = head
    billboard.Size = UDim2.fromOffset(180, 54)
    billboard.StudsOffsetWorldSpace = Vector3.new(0, 3.15, 0)
    billboard.AlwaysOnTop = true
    billboard.MaxDistance = 85
    billboard.LightInfluence = 0
    billboard.ResetOnSpawn = false
    billboard.Parent = playerGui

    local holder = Instance.new("Frame")
    holder.Size = UDim2.fromScale(1, 1)
    holder.BackgroundTransparency = 1
    holder.Parent = billboard

    local badge = Instance.new("Frame")
    badge.Name = "Badge"
    badge.AnchorPoint = Vector2.new(0.5, 0)
    badge.Position = UDim2.new(0.5, 0, 0, 0)
    badge.Size = UDim2.fromOffset(76, 20)
    badge.BackgroundColor3 = Color3.fromRGB(23, 15, 31)
    badge.BackgroundTransparency = 0.08
    badge.BorderSizePixel = 0
    badge.Parent = holder

    lHubCorner(badge, 8)

    local badgeStroke = lHubStroke(
        badge,
        Color3.fromRGB(172, 78, 255),
        1.5,
        0.05
    )

    local hubText = Instance.new("TextLabel")
    hubText.Name = "HubLabel"
    hubText.Size = UDim2.fromScale(1, 1)
    hubText.BackgroundTransparency = 1
    hubText.Text = "L HUB"
    hubText.TextColor3 = Color3.fromRGB(226, 168, 255)
    hubText.TextStrokeColor3 = Color3.fromRGB(55, 16, 76)
    hubText.TextStrokeTransparency = 0.15
    hubText.Font = Enum.Font.GothamBlack
    hubText.TextSize = 12
    hubText.Parent = badge

    local shadow = Instance.new("TextLabel")
    shadow.Name = "NameShadow"
    shadow.AnchorPoint = Vector2.new(0.5, 0)
    shadow.Position = UDim2.new(0.5, 1, 0, 27)
    shadow.Size = UDim2.new(1, -8, 0, 25)
    shadow.BackgroundTransparency = 1
    shadow.Text = player.DisplayName
    shadow.TextColor3 = Color3.fromRGB(20, 7, 29)
    shadow.TextTransparency = 0.08
    shadow.Font = Enum.Font.GothamBlack
    shadow.TextSize = 19
    shadow.TextXAlignment = Enum.TextXAlignment.Center
    shadow.Parent = holder

    local displayName = Instance.new("TextLabel")
    displayName.Name = "DisplayName"
    displayName.AnchorPoint = Vector2.new(0.5, 0)
    displayName.Position = UDim2.new(0.5, 0, 0, 26)
    displayName.Size = UDim2.new(1, -8, 0, 25)
    displayName.BackgroundTransparency = 1
    displayName.Text = player.DisplayName
    displayName.TextColor3 = Color3.fromRGB(255, 255, 255)
    displayName.TextStrokeColor3 = Color3.fromRGB(77, 25, 105)
    displayName.TextStrokeTransparency = 0.05
    displayName.Font = Enum.Font.GothamBlack
    displayName.TextSize = 19
    displayName.TextXAlignment = Enum.TextXAlignment.Center
    displayName.Parent = holder

    task.spawn(function()
        local colors = {
            Color3.fromRGB(255, 255, 255),
            Color3.fromRGB(236, 192, 255),
            Color3.fromRGB(255, 153, 223),
            Color3.fromRGB(196, 151, 255),
        }

        local index = 1

        while billboard.Parent
            and character.Parent
            and State.Running
        do
            index = (index % #colors) + 1
            local target = colors[index]

            local nameTween = TweenService:Create(
                displayName,
                TweenInfo.new(0.48, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
                {
                    TextColor3 = target,
                    TextTransparency = 0,
                }
            )

            local hubTween = TweenService:Create(
                hubText,
                TweenInfo.new(0.48, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
                {
                    TextColor3 = target,
                }
            )

            local borderTween = TweenService:Create(
                badgeStroke,
                TweenInfo.new(0.48, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
                {
                    Color = target,
                    Transparency = 0.03,
                }
            )

            nameTween:Play()
            hubTween:Play()
            borderTween:Play()

            nameTween.Completed:Wait()
            task.wait(0.18)
        end
    end)
end

LHubCharacterConnection = player.CharacterAdded:Connect(function(character)
    task.defer(createLHubTag, character)
end)

if player.Character then
    task.defer(createLHubTag, player.Character)
end

local AutomationLoopRunning = true
local AutomationLoopInterval = 0.05

task.spawn(function()
    while State.Running and AutomationLoopRunning do
        local now = os.clock()
        local menuOpen = isCoinClickerMenuOpen(now)

        safetyCheck(now)
        setActivity(menuOpen)
        handleMenuTransition(menuOpen, now)

        -- IMPORTANTE:
        -- Não usamos "return" quando o menu fecha ou a proteção pausa.
        -- "return" encerrava o loop inteiro e nenhuma automação voltava a funcionar.
        if menuOpen and not Safety.Paused then
            if State.CompactView
                and compactReapplyAt > 0
                and now >= compactReapplyAt
            then
                compactReapplyAt = 0
                pcall(applyCompact)
            end

            if State.AutoItems
                and now - State.LastItems >= State.ItemInterval
            then
                State.LastItems = now

                local bought = buyBestGenerator()

                if not bought then
                    State.LastItems = now + 0.75
                end
            end

            if State.AutoBuff
                and now - State.LastBuff >= State.BuffInterval
            then
                State.LastBuff = now
                buyAvailableUpgrade()
            end

            if State.AutoWrinkler
                and now - State.LastWrinkler >= State.WrinklerInterval
            then
                State.LastWrinkler = now
                popWrinklers()
            end

            if State.AutoGolden
                and now - State.LastGolden >= State.GoldenInterval
            then
                State.LastGolden = now
                clickGoldens()
            end

            if State.AutoBlackMarket
                and now - State.LastBlackMarket >= State.BlackMarketInterval
            then
                State.LastBlackMarket = now
                stepBlackMarket(now)
            end

            if State.CompactView
                and now - State.LastCompact >= 1.25
            then
                State.LastCompact = now

                if CompactFortunesButton
                    and CompactFortunesButton.Visible
                then
                    local left = getFrame("LeftColumn")

                    if left then
                        local camera = workspace.CurrentCamera
                        local viewport = camera and camera.ViewportSize

                        local buttonHeight =
                            CompactFortunesButton.AbsoluteSize.Y > 0
                            and CompactFortunesButton.AbsoluteSize.Y
                            or (isMobile and 38 or 36)

                        local availableWidth = math.max(
                            130,
                            math.floor(left.AbsoluteSize.X - 12)
                        )

                        local buttonWidth = math.min(
                            isMobile and 210 or 196,
                            availableWidth
                        )

                        local x = math.floor(
                            left.AbsolutePosition.X
                            + (left.AbsoluteSize.X - buttonWidth) / 2
                        )

                        local y = math.floor(
                            left.AbsolutePosition.Y
                            + left.AbsoluteSize.Y
                            + 6
                        )

                        if viewport then
                            y = math.min(
                                y,
                                viewport.Y - buttonHeight - 12
                            )
                        end

                        CompactFortunesButton.Size =
                            UDim2.fromOffset(buttonWidth, buttonHeight)

                        CompactFortunesButton.Position =
                            UDim2.fromOffset(x, y)
                    end
                end
            end
        end

        task.wait(AutomationLoopInterval)
    end
end)

setActivity(isCoinClickerMenuOpen())

local Minimized = false
local FullSize = Main.Size

local FullTitleSize = Title.Size
local FullTitleTextSize = Title.TextSize

local function setMinimized(value)
    Minimized = value

    tween(HubScale, TweenInfo.new(0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
        Scale = 0.985
    })

    task.delay(0.07, function()
        if HubScale and HubScale.Parent then
            tween(HubScale, TweenInfo.new(0.13, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Scale = 1
            })
        end
    end)

    if Minimized then
        Body.Visible = false
        Activity.Visible = false
        CreditWrap.Visible = false

        Title.Size = UDim2.new(1, -82, 1, 0)
        Title.TextSize = isMobile and 11 or 12

        tween(Main, TweenInfo.new(0.16, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
            Size = UDim2.fromOffset(isMobile and 238 or 255, 36)
        })

        Minimize.Text = "+"
    else
        Body.Visible = true
        Activity.Visible = true
        CreditWrap.Visible = true

        Title.Size = FullTitleSize
        Title.TextSize = FullTitleTextSize

        tween(Main, TweenInfo.new(0.18, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
            Size = FullSize
        })

        Minimize.Text = "—"
    end
end

Minimize.Activated:Connect(function()
    setMinimized(not Minimized)
end)

local function stop()
    State.Running = false
    State.CompactView = false

    if LHubCharacterConnection then
        LHubCharacterConnection:Disconnect()
        LHubCharacterConnection = nil
    end

    clearLHubTag()
    pcall(applyCompact)

    if CompactFortunesButton then
        CompactFortunesButton.Visible = false
    end

    AutomationLoopRunning = false

    if Gui then
        Gui:Destroy()
    end

    if SoundFolder then
        SoundFolder:Destroy()
    end
end

Close.Activated:Connect(function()
    tween(HubScale, TweenInfo.new(0.14, Enum.EasingStyle.Quart, Enum.EasingDirection.In), {
        Scale = 0.92
    })

    tween(Main, TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
        BackgroundTransparency = 1,
        Position = Main.Position + UDim2.fromOffset(0, 8)
    })

    task.delay(0.13, function()
        stop()
    end)
end)

print("[CoinClicker Hub] carregado.")
