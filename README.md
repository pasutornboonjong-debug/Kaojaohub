--[[
IMS Block Spin Main v17 — Aimbot Exclusion + Crate Animation Skip + Stable ESP
[FIXED] ใช้สคริปต์หลัก v13 ที่ผู้ใช้อัปโหลดเป็นฐาน
[FIXED] นำชุด Auto-Farm / เส้นทางรถ / Fishing Farm และแท็บ Fish ออกจากรุ่นนี้ตามคำขอ
[FIXED] คง Main, Player ESP, Combat, Movement, General, Settings และระบบเสริมที่ไม่เกี่ยวกับ Auto-Farm ไว้
หมายเหตุ: Aimbot รุ่นนี้เป็น Camera Aim Assist ล็อกเป้าภายใน FOV ไม่เปลี่ยน Remote หรือทิศทางกระสุนฝั่งเซิร์ฟเวอร์
]]

--[[
  IMS Loader v20 — หน้าเลือกอุปกรณ์ (โทรศัพท์ / คอม) + สแกน "ทุกระบบ" ของเกมก่อนเริ่มใช้งาน
  วิธีใช้: วางไว้ "บนสุดของไฟล์สคริปต์หลัก" (ก่อนบรรทัดแรกเดิม)
  ขั้นตอน: 1) เลือกอุปกรณ์ (มีค่าที่จำจากครั้งก่อนเลือกให้ ถ้าไม่แตะอะไรจะใช้ค่านั้นเมื่อครบ 8 วินาที)
           2) สแกนระบบทั้งหมดแบบอ่านอย่างเดียว (ไม่ยิงรีโมต ไม่แก้ค่าอะไร) ไม่ว่าเราจะเปิดฟีเจอร์นั้นไว้หรือไม่
           3) ขึ้น "เสร็จสิ้น" แล้วสคริปต์หลักทำงานต่อ
  ผลลัพธ์: getgenv().IMS_Platform = "mobile" | "pc"      getgenv().IMS_Scan = ผลสแกนทั้งหมด
           getgenv().IMS_LoaderError = ข้อความผิดพลาด (ถ้ามี)
  ทำงานในเธรดแยกและมีเวลาสูงสุดสำหรับการสแกน จึงไม่ทำให้สคริปต์หลักค้างหรือไม่ขึ้น

  ต้องแก้ 1 บรรทัดในสคริปต์หลักให้ใช้ค่าที่เลือก: หาบรรทัด
      local IS_MOBILE = UserInputService.TouchEnabled
  แล้วเปลี่ยนเป็น
      local IS_MOBILE = (getgenv().IMS_Platform == "mobile")
          or (getgenv().IMS_Platform == nil and UserInputService.TouchEnabled)
]]
do
    local Players = game:GetService("Players")
    local ReplicatedStorage = game:GetService("ReplicatedStorage")
    local TweenService = game:GetService("TweenService")
    local Workspace = game:GetService("Workspace")
    local UserInputService = game:GetService("UserInputService")

    local activeGuis = {}
    local skipped = false

    local function closeAllGuis()
        for _, g in ipairs(activeGuis) do pcall(function() g:Destroy() end) end
        table.clear(activeGuis)
    end

    local function makeGui(name, order)
        local gui = Instance.new("ScreenGui")
        gui.Name = name
        gui.ResetOnSpawn = false
        gui.IgnoreGuiInset = true
        gui.DisplayOrder = order
        pcall(function() gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling end)
        local parent
        pcall(function() if gethui then parent = gethui() end end)
        if not parent then pcall(function() parent = game:GetService("CoreGui") end) end
        if parent then pcall(function() gui.Parent = parent end) end
        if not gui.Parent then gui.Parent = Players.LocalPlayer:WaitForChild("PlayerGui") end
        table.insert(activeGuis, gui)
        return gui
    end

    local function makeScale()
        local s = Instance.new("UIScale")
        pcall(function()
            local cam = Workspace.CurrentCamera
            local vp = cam and cam.ViewportSize or Vector2.new(800, 400)
            s.Scale = math.clamp(math.min(vp.X / 400, vp.Y / 470), 0.55, 1)
        end)
        return s
    end

    local function readFile(path)
        local value
        pcall(function()
            if isfile and isfile(path) then value = tostring(readfile(path)):gsub("%s", "") end
        end)
        return value
    end

    local function writeFile(path, text)
        pcall(function()
            if makefolder and isfolder and not isfolder("ItemManagementSystem") then
                makefolder("ItemManagementSystem")
            end
            if writefile then writefile(path, text) end
        end)
    end

    local lang = (readFile("ItemManagementSystem/lang.txt") == "en") and "en" or "th"
    local TXT = {
        pickTitle = { th = "เลือกอุปกรณ์ของคุณ", en = "Choose your device" },
        phone     = { th = "โทรศัพท์", en = "Phone" },
        pc        = { th = "คอม", en = "PC" },
        auto      = { th = "ไม่เลือก = ใช้ค่าเดิมอัตโนมัติใน", en = "No choice = auto-continue in" },
        sec       = { th = "วินาที", en = "s" },
        last      = { th = "ครั้งก่อนเลือก", en = "last time" },
        title     = { th = "กำลังโหลดระบบ", en = "Loading system" },
        sub       = { th = "ตรวจทุกระบบของเกมก่อนเริ่มใช้งาน (ไม่ว่าจะเปิดฟีเจอร์ไว้หรือไม่)", en = "Checking every game system before you start" },
        done      = { th = "เสร็จสิ้น — พร้อมใช้งาน", en = "Done — ready to use" },
        skip      = { th = "ข้าม", en = "Skip" },
        ready     = { th = "ระบบที่พร้อมใช้", en = "Systems ready" },
        missing   = { th = "ไม่พบ", en = "missing" },
        game      = { th = "เกมและตัวละคร", en = "Game and character" },
        ui        = { th = "ไลบรารีหน้าต่าง (Fluent)", en = "Window library (Fluent)" },
        items     = { th = "ข้อมูลไอเทม (Items)", en = "Item data (Items)" },
        fish      = { th = "ข้อมูลปลา (Items.fish)", en = "Fish data (Items.fish)" },
        drop      = { th = "ไอเทมที่ตกพื้น (drop item)", en = "Dropped items (drop item)" },
        sprint    = { th = "ระบบสเตมิน่า (Sprint)", en = "Stamina system (Sprint)" },
        camera    = { th = "กล้อง/แรงสะท้อน (ShoulderCamera)", en = "Camera/recoil (ShoulderCamera)" },
        gun       = { th = "ปืนที่มีค่า Recoil", en = "Guns with Recoil" },
        fishing   = { th = "ระบบตกปลา (มินิเกม)", en = "Fishing minigame" },
        crate     = { th = "ระบบตู้สุ่ม (Crate)", en = "Crate system" },
        vehicle   = { th = "ระบบรถ (VehicleSeat)", en = "Vehicles (VehicleSeat)" },
        render    = { th = "การแสดงผล (Highlight/Billboard)", en = "Rendering (Highlight/Billboard)" },
        exec      = { th = "ความสามารถของ executor", en = "Executor features" },
    }
    local function L(key) return TXT[key][lang] end

    ------------------------------------------------------------------
    -- 1) หน้าเลือกอุปกรณ์
    ------------------------------------------------------------------
    local function choosePlatform()
        local saved = readFile("ItemManagementSystem/platform.txt")
        if saved ~= "mobile" and saved ~= "pc" then saved = nil end
        local default = saved or ((UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled) and "mobile" or "pc")
        local chosen

        local ok = pcall(function()
            local gui = makeGui("IMS_DevicePicker", 1001)

            local dim = Instance.new("Frame")
            dim.Size = UDim2.fromScale(1, 1)
            dim.BackgroundColor3 = Color3.new(0, 0, 0)
            dim.BackgroundTransparency = 0.35
            dim.BorderSizePixel = 0
            dim.Parent = gui

            local card = Instance.new("Frame")
            card.AnchorPoint = Vector2.new(0.5, 0.5)
            card.Position = UDim2.fromScale(0.5, 0.5)
            card.Size = UDim2.fromOffset(340, 230)
            card.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
            card.Parent = gui
            makeScale().Parent = card
            local cc = Instance.new("UICorner")
            cc.CornerRadius = UDim.new(0, 14)
            cc.Parent = card
            local cs = Instance.new("UIStroke")
            cs.Color = Color3.fromRGB(120, 170, 255)
            cs.Thickness = 2
            cs.Parent = card

            local title = Instance.new("TextLabel")
            title.BackgroundTransparency = 1
            title.Size = UDim2.new(1, 0, 0, 46)
            title.Font = Enum.Font.GothamBold
            title.TextSize = 19
            title.TextColor3 = Color3.new(1, 1, 1)
            title.Text = L("pickTitle")
            title.Parent = card

            local function choice(text, value, xScale, xOffset)
                local b = Instance.new("TextButton")
                b.Size = UDim2.new(0.5, -22, 0, 76)
                b.Position = UDim2.new(xScale, xOffset, 0, 56)
                local isDefault = (value == default)
                b.BackgroundColor3 = isDefault and Color3.fromRGB(60, 110, 200) or Color3.fromRGB(44, 44, 54)
                b.TextColor3 = Color3.new(1, 1, 1)
                b.Font = Enum.Font.GothamBold
                b.TextSize = 18
                b.Text = text .. (isDefault and saved and ("\n(" .. L("last") .. ")") or "")
                b.Parent = card
                local c = Instance.new("UICorner")
                c.CornerRadius = UDim.new(0, 10)
                c.Parent = b
                b.Activated:Connect(function() chosen = value end)
            end
            choice(L("phone"), "mobile", 0, 16)
            choice(L("pc"), "pc", 0.5, 6)

            local hint = Instance.new("TextLabel")
            hint.BackgroundTransparency = 1
            hint.Position = UDim2.new(0, 12, 0, 148)
            hint.Size = UDim2.new(1, -24, 0, 64)
            hint.Font = Enum.Font.Gotham
            hint.TextSize = 13
            hint.TextWrapped = true
            hint.TextColor3 = Color3.fromRGB(170, 170, 185)
            hint.Parent = card

            local total = 8
            for left = total, 1, -1 do
                if chosen then break end
                hint.Text = L("auto") .. " " .. left .. " " .. L("sec")
                task.wait(1)
            end
        end)
        if not ok then warn("[IMS Loader] picker failed, using default") end

        chosen = chosen or default
        closeAllGuis()
        writeFile("ItemManagementSystem/platform.txt", chosen)
        getgenv().IMS_Platform = chosen
        return chosen
    end

    local okPick, errPick = pcall(choosePlatform)
    if not okPick then
        warn("[IMS Loader] platform error:", errPick)
        closeAllGuis()
        getgenv().IMS_Platform = getgenv().IMS_Platform or (UserInputService.TouchEnabled and "mobile" or "pc")
    end

    ------------------------------------------------------------------
    -- 2) สแกนทุกระบบ (อ่านอย่างเดียว)
    ------------------------------------------------------------------
    local function runLoader()
        if not game:IsLoaded() then game.Loaded:Wait() end
        local LocalPlayer = Players.LocalPlayer
        while not LocalPlayer do
            task.wait(0.1)
            LocalPlayer = Players.LocalPlayer
        end

        local gui = makeGui("IMS_Loading", 1000)

        local dim = Instance.new("Frame")
        dim.Size = UDim2.fromScale(1, 1)
        dim.BackgroundColor3 = Color3.new(0, 0, 0)
        dim.BackgroundTransparency = 0.35
        dim.BorderSizePixel = 0
        dim.Parent = gui

        local card = Instance.new("Frame")
        card.AnchorPoint = Vector2.new(0.5, 0.5)
        card.Position = UDim2.fromScale(0.5, 0.5)
        card.Size = UDim2.fromOffset(380, 440)
        card.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
        card.Parent = gui
        makeScale().Parent = card
        local cardCorner = Instance.new("UICorner")
        cardCorner.CornerRadius = UDim.new(0, 14)
        cardCorner.Parent = card
        local cardStroke = Instance.new("UIStroke")
        cardStroke.Color = Color3.fromRGB(120, 170, 255)
        cardStroke.Thickness = 2
        cardStroke.Parent = card

        local function label(text, y, h, size, bold, color)
            local l = Instance.new("TextLabel")
            l.BackgroundTransparency = 1
            l.Position = UDim2.new(0, 16, 0, y)
            l.Size = UDim2.new(1, -32, 0, h)
            l.Font = bold and Enum.Font.GothamBold or Enum.Font.Gotham
            l.TextSize = size
            l.TextColor3 = color or Color3.new(1, 1, 1)
            l.TextXAlignment = Enum.TextXAlignment.Left
            l.TextWrapped = true
            l.Text = text
            l.Parent = card
            return l
        end

        local titleLabel = label(L("title"), 12, 26, 20, true)
        local subLabel = label(L("sub"), 40, 32, 12, false, Color3.fromRGB(170, 170, 185))
        local stepLabel = label("...", 76, 20, 14, true, Color3.fromRGB(120, 170, 255))

        local barBg = Instance.new("Frame")
        barBg.Position = UDim2.new(0, 16, 0, 102)
        barBg.Size = UDim2.new(1, -32, 0, 12)
        barBg.BackgroundColor3 = Color3.fromRGB(44, 44, 56)
        barBg.BorderSizePixel = 0
        barBg.Parent = card
        local bgCorner = Instance.new("UICorner")
        bgCorner.CornerRadius = UDim.new(1, 0)
        bgCorner.Parent = barBg
        local fill = Instance.new("Frame")
        fill.Size = UDim2.new(0, 0, 1, 0)
        fill.BackgroundColor3 = Color3.fromRGB(80, 200, 120)
        fill.BorderSizePixel = 0
        fill.Parent = barBg
        local fillCorner = Instance.new("UICorner")
        fillCorner.CornerRadius = UDim.new(1, 0)
        fillCorner.Parent = fill

        local pctLabel = label("0%", 118, 16, 12, false, Color3.fromRGB(170, 170, 185))
        pctLabel.TextXAlignment = Enum.TextXAlignment.Right

        local log = label("", 140, 260, 13, false)
        log.RichText = true
        log.TextYAlignment = Enum.TextYAlignment.Top

        local skip = Instance.new("TextButton")
        skip.AnchorPoint = Vector2.new(1, 1)
        skip.Position = UDim2.new(1, -12, 1, -10)
        skip.Size = UDim2.fromOffset(64, 24)
        skip.BackgroundColor3 = Color3.fromRGB(44, 44, 56)
        skip.TextColor3 = Color3.new(1, 1, 1)
        skip.Font = Enum.Font.GothamBold
        skip.TextSize = 12
        skip.Text = L("skip")
        skip.Parent = card
        local skipCorner = Instance.new("UICorner")
        skipCorner.CornerRadius = UDim.new(0, 8)
        skipCorner.Parent = skip
        skip.Activated:Connect(function() skipped = true end)

        local lines = {}
        local function addLine(ok, name, detail)
            local color = ok and "#50C878" or "#FF6B6B"
            table.insert(lines, string.format('<font color="%s">%s</font> %s%s', color, ok and "✓" or "✗", name,
                detail and ("  <font color=\"#9aa0b5\">" .. detail .. "</font>") or ""))
            if #lines > 15 then table.remove(lines, 1) end
            log.Text = table.concat(lines, "\n")
        end

        local function setProgress(p, text)
            p = math.clamp(p, 0, 1)
            stepLabel.Text = text
            pctLabel.Text = math.floor(p * 100) .. "%"
            TweenService:Create(fill, TweenInfo.new(0.25, Enum.EasingStyle.Quad), { Size = UDim2.new(p, 0, 1, 0) }):Play()
        end

        local function waitFor(fn, timeout)
            local t = 0
            while t < timeout and not skipped do
                local ok, res = pcall(fn)
                if ok and res then return res end
                task.wait(0.1)
                t = t + 0.1
            end
            return nil
        end

        local function path(root, ...)
            local node = root
            for _, name in ipairs({ ... }) do
                node = node and node:FindFirstChild(name)
            end
            return node
        end

        local function requireModule(...)
            local inst = path(ReplicatedStorage, ...)
            if not inst then return nil end
            local ok, mod = pcall(require, inst)
            if ok and type(mod) == "table" then return mod end
            return nil
        end

        local scan = { Ready = {}, Counts = {} }
        getgenv().IMS_Scan = scan

        local steps = {
            { "game", function()
                local char = waitFor(function() return LocalPlayer.Character end, 5)
                local hum = char and waitFor(function() return char:FindFirstChildOfClass("Humanoid") end, 3)
                local backpack = LocalPlayer:FindFirstChild("Backpack") ~= nil
                local others = #Players:GetPlayers() - 1
                scan.Counts.Players = others
                return hum ~= nil and backpack, string.format("PlaceId %s | %d players", tostring(game.PlaceId), others + 1)
            end },
            { "ui", function()
                -- ทดสอบดาวน์โหลดไลบรารีหน้าต่าง (แค่ดึงข้อความ ไม่รันโค้ด) ถ้าไม่ผ่าน = สคริปต์หลักจะไม่มีหน้าต่างขึ้น
                local url = "https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"
                local ok, body = pcall(function() return game:HttpGet(url) end)
                if not ok then
                    scan.UIError = tostring(body)
                    return false, "download failed"
                end
                local size = type(body) == "string" and #body or 0
                scan.Counts.FluentBytes = size
                return size > 1000, size .. " bytes"
            end },
            { "items", function()
                local items = waitFor(function() return ReplicatedStorage:FindFirstChild("Items") end, 3)
                if not items then return false end
                local folders, tools = 0, 0
                for _, folder in ipairs(items:GetChildren()) do
                    folders = folders + 1
                    for _, t in ipairs(folder:GetChildren()) do
                        if t:IsA("Tool") then tools = tools + 1 end
                    end
                end
                scan.Counts.ItemFolders, scan.Counts.Tools = folders, tools
                return tools > 0, string.format("%d folders | %d items", folders, tools)
            end },
            { "fish", function()
                local folder = path(ReplicatedStorage, "Items", "fish")
                local n, withRarity = 0, 0
                if folder then
                    for _, t in ipairs(folder:GetChildren()) do
                        if t:IsA("Tool") then
                            n = n + 1
                            if t:GetAttribute("RarityName") ~= nil then withRarity = withRarity + 1 end
                        end
                    end
                end
                scan.Counts.Fish = n
                return n > 0, string.format("%d fish | %d with rarity", n, withRarity)
            end },
            { "drop", function()
                local folder = Workspace:FindFirstChild("drop item")
                if not folder then return false, "folder not loaded yet" end
                local zones = 0
                for _, d in ipairs(folder:GetDescendants()) do
                    if d:IsA("BasePart") and string.find(string.lower(d.Name), "pickup", 1, true) then zones = zones + 1 end
                end
                scan.Counts.PickupZones = zones
                return true, zones .. " pickup zones"
            end },
            { "sprint", function()
                local m = waitFor(function() return requireModule("Modules", "Game", "Sprint") end, 2)
                return m ~= nil and type(m.sprint_bar) == "table"
            end },
            { "camera", function()
                local m = waitFor(function() return requireModule("Modules", "Game", "ShoulderCamera") end, 2)
                return m ~= nil and m.recoil ~= nil
            end },
            { "gun", function()
                local guns = path(ReplicatedStorage, "Items", "gun")
                local withRecoil = 0
                if guns then
                    for _, t in ipairs(guns:GetChildren()) do
                        if t:IsA("Tool") and type(t:GetAttribute("Recoil")) == "number" then withRecoil = withRecoil + 1 end
                    end
                end
                scan.Counts.GunsWithRecoil = withRecoil
                return withRecoil > 0, withRecoil .. " guns"
            end },
            { "fishing", function()
                local m = waitFor(function() return requireModule("Modules", "Game", "Minigames", "SliderConstants") end, 2)
                local okMod = m ~= nil and type(m.needle_at) == "function"
                local pg = LocalPlayer:FindFirstChild("PlayerGui")
                local frame = pg and pg:FindFirstChild("SliderMinigameFrame", true)
                scan.Counts.MinigameUI = frame ~= nil
                return okMod, (okMod and "formula ok" or nil) .. (frame and " | UI ok" or " | UI loads on bite")
            end },
            { "crate", function()
                local ui = path(ReplicatedStorage, "Modules", "Game", "CrateSystem", "CrateUI")
                local crate = path(ReplicatedStorage, "Modules", "Game", "CrateSystem", "Crate")
                return ui ~= nil and crate ~= nil
            end },
            { "vehicle", function()
                local seat = Workspace:FindFirstChildWhichIsA("VehicleSeat", true)
                scan.Counts.VehicleSeat = seat ~= nil
                return true, seat and "seat found" or "spawn on demand"
            end },
            { "render", function()
                local okH = pcall(function() Instance.new("Highlight"):Destroy() end)
                local okB = pcall(function() Instance.new("BillboardGui"):Destroy() end)
                return okH and okB, (okH and "Highlight ok" or "no Highlight") .. " | " .. (okB and "Billboard ok" or "no Billboard")
            end },
            { "exec", function()
                local feats = {
                    { "files", writefile ~= nil and readfile ~= nil },
                    { "asset", (getcustomasset or getsynasset) ~= nil },
                    { "http", (request or http_request or (syn and syn.request)) ~= nil },
                    { "touch", firetouchinterest ~= nil },
                    { "prompt", fireproximityprompt ~= nil },
                    { "conns", getconnections ~= nil },
                    { "gethui", gethui ~= nil },
                    { "fpscap", setfpscap ~= nil },
                    { "vim", pcall(function() return game:GetService("VirtualInputManager") end) },
                }
                local have, names = 0, {}
                for _, f in ipairs(feats) do
                    scan.Ready["exec_" .. f[1]] = f[2] and true or false
                    if f[2] then have = have + 1 else table.insert(names, f[1]) end
                end
                local detail = string.format("%d/%d", have, #feats)
                if #names > 0 then detail = detail .. " (no: " .. table.concat(names, ", ") .. ")" end
                return have >= 5, detail
            end },
        }

        for i, step in ipairs(steps) do
            if skipped then break end
            local key = step[1]
            setProgress((i - 1) / #steps, L(key) .. " ...")
            local ok, res, detail = pcall(step[2])
            local passed = ok and res and true or false
            scan.Ready[key] = passed
            local text
            if ok and detail then
                text = detail
            elseif not passed then
                text = L("missing")
            end
            addLine(passed, L(key), text)
            setProgress(i / #steps, L(key))
            task.wait(0.15)
        end

        local readyCount, total = 0, 0
        for _, step in ipairs(steps) do
            total = total + 1
            if scan.Ready[step[1]] then readyCount = readyCount + 1 end
        end

        setProgress(1, L("done"))
        stepLabel.TextColor3 = Color3.fromRGB(80, 200, 120)
        titleLabel.Text = L("ready") .. ": " .. readyCount .. "/" .. total
        task.wait(1.6)
        closeAllGuis()
    end

    -- รันในเธรดแยก รอสูงสุด 14 วินาที: ต่อให้ค้าง/พัง สคริปต์หลักก็ยังทำงานต่อได้เสมอ
    local finished = false
    task.spawn(function()
        local ok, err = pcall(runLoader)
        if not ok then
            warn("[IMS Loader] error:", err)
            getgenv().IMS_LoaderError = tostring(err)
        end
        finished = true
    end)

    local t0 = os.clock()
    while not finished and os.clock() - t0 < 14 do
        task.wait(0.1)
    end
    if not finished then
        skipped = true
        warn("[IMS Loader] timeout 14s — continuing without waiting")
        getgenv().IMS_LoaderError = "timeout"
    end
    closeAllGuis()
end


--[[
    Item Management System (Fluent UI) - รุ่น v12 (ระบบภาษา ไทย/English, มุดดิน, จำค่าอัตโนมัติ)
    ------------------------------------------------------------
    โครงสร้างข้อมูลเกม:
      workspace["drop item"]        -> โฟลเดอร์ไอเทมที่ตกพื้น
        └─ Model                    -> ตัวไอเทม
             └─ "PickUp Zone"       -> โซนเก็บไอเทม

    ระบบ:
      1) Auto-Collect โหมด Pull : ดูด "ทุกชิ้นในระยะพร้อมกัน" ด้วยลูปเดียวทุกเฟรม
                                  จนกว่าไอเทมจะหายจากพื้น (เข้ากระเป๋า)
      2) Auto-Collect โหมด Walk : เช็คระยะก่อน -> เดินไปเก็บ -> เดินกลับจุดเดิม
      3) ESP                    : ชื่อ | ความแรร์ | ระยะ (ลบทันทีเมื่อถูกเก็บ)
      4) Player ESP             : แสดงไอเทมในตัวผู้เล่นคนอื่น สีตามความแรร์ (ปรับระยะ/ตัวกรองได้)
      5) Combat                 : ไม่มีแรงสะท้อนปืน (จอไม่สั่น) + สเตมิน่าไม่จำกัด + ตรวจจับระบบเกมก่อนใช้
            5c) Discord               : แจ้งเตือนเข้า Discord เมื่อตัวละครเราตาย
      5b) General               : ระยะ/ความเร็วดูด, สี ESP, ขนาด GUI, ลดกราฟิก, FPS + ล็อก FPS
      5e) Fish                  : ติดตามปลาที่ตกได้ตามระดับ + เตือนเมื่อปลาที่ควรทิ้งสะสม (ไม่ทิ้งให้เอง)
      5d) Movement              : มุดดิน (คอมกด Z / มือถือปุ่ม Snap) ปรับความลึกได้
      6) Settings               : ธีม/คอนฟิก (InterfaceManager + SaveManager ของ Fluent)
    ทุกส่วนครอบด้วย pcall / task.spawn เพื่อไม่ให้สคริปต์หยุดทำงานเมื่อเกิด Error
]]

--// ============================== ค่าคงที่ (แก้ในโค้ด) ==============================
local CONFIG = {
    DROP_FOLDER_NAME = "drop item",   -- ชื่อโฟลเดอร์ไอเทม (ไม่สนตัวพิมพ์/เว้นวรรค)
    ZONE_NAME        = "PickUp Zone", -- ชื่อโซนเก็บไอเทม
    SCAN_INTERVAL    = 0.20,           -- ความถี่สแกนหาไอเทม (วินาที)
    ESP_INTERVAL     = 0.45,           -- ความถี่อัปเดต ESP (วินาที)
    PULL_TRIGGER     = 0.12,          -- ความถี่กระตุ้นการเก็บของทุกชิ้นระหว่างดูด (วินาที)
    PULL_CLOSE       = 1.5,           -- (เมื่อความเร็ว<1) ใกล้กว่านี้ให้วางทับตัวเลย
    WALK_TIMEOUT     = 2.5,           -- เวลารอสูงสุดต่อ 1 waypoint (วินาที)
    SKIP_TIME        = 5,             -- พักไอเทมที่โหมด Walk เก็บไม่สำเร็จ (วินาที)
    FALLBACK_REFRESH = 5,             -- ความถี่ค้นหาสำรองทั้งแมพ (วินาที)
    NOTIFY_BATCH     = 0.5,           -- รวมแจ้งเตือนการเก็บเป็นชุด ทุกกี่วินาที

    -- Discord Webhook (ใส่ URL ของ webhook ตัวเอง)
    DISCORD_WEBHOOK_URL = "",
}

--// ============================== Services ==============================
local Players            = game:GetService("Players")
local PathfindingService = game:GetService("PathfindingService")
local RunService         = game:GetService("RunService")
local Lighting           = game:GetService("Lighting")
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local Workspace          = game:GetService("Workspace")
local HttpService        = game:GetService("HttpService")
local UserInputService   = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
-- อุปกรณ์ที่เลือกจากหน้าโหลด (IMS_Loader) ถ้าไม่มีให้เดาจากหน้าจอสัมผัส
local IS_MOBILE = (getgenv().IMS_Platform == "mobile")
    or (getgenv().IMS_Platform == nil and UserInputService.TouchEnabled)

--// ============================== ภาษา (ไทย / English) ==============================
-- ข้อความในสคริปต์เขียนเป็นภาษาไทย ถ้าเลือก English จะแปลตอนแสดงผล (UI, แจ้งเตือน, Discord)
-- ภาษาที่เลือกถูกจำไว้ในไฟล์ ItemManagementSystem/lang.txt ของเครื่องนี้
local Ext = {}
local T

do
    local LANG_FILE = "ItemManagementSystem/lang.txt"
    local PAIRS = {
        { "Webhook ถูกลบหรือ URL ผิด", "Webhook deleted or URL is wrong" },
        { "Webhook ไม่ถูกต้อง", "Invalid webhook" },
        { "ส่งถี่เกิน รอสักครู่", "Rate limited, wait a moment" },
        { "ชิ้น", "items" },
        { "เช่น ", "e.g. " },
        { "1.00 = ดูดทันทีทั้งหมด, ต่ำกว่านี้ = ค่อย ๆ ดูด", "1.00 = pull everything instantly, lower = pull gradually" },
        { "Discord User ID ที่จะให้แท็ก (ไม่ใส่ก็ได้)", "Discord User ID to ping (optional)" },
        { "FPS Lock: executor ไม่รองรับ setfpscap", "FPS Lock: executor does not support setfpscap" },
        { "FPS Lock: ตั้งค่าไม่สำเร็จ", "FPS Lock: failed to apply" },
        { "FPS Lock: ปิด", "FPS Lock: off" },
        { "FPS Lock: ไม่รองรับใน executor นี้", "FPS Lock: not supported by this executor" },
        { "ระบบสเตมิน่า (Sprint)", "Stamina system (Sprint)" },
        { "กล้อง/แรงสะท้อน (ShoulderCamera)", "Camera/recoil (ShoulderCamera)" },
        { "ปืนที่พบในตัว (มี Recoil)", "Guns found on you (with Recoil)" },
        { "Pull = ดูดทุกชิ้นพร้อมกัน | Walk = เดินไปเก็บทีละชิ้นแล้วกลับที่เดิม", "Pull = vacuum everything at once | Walk = walk to each item, then return" },
        { "URL ไม่ถูกต้อง (ต้องขึ้นต้นด้วย https://discord.com/api/webhooks/...)", "Invalid URL (must start with https://discord.com/api/webhooks/...)" },
        { "Ultra Low ภาพกากมาก แต่ลื่นสุด | เลือก Off เพื่อคืนค่าเดิม", "Ultra Low looks very rough but runs smoothest | choose Off to restore" },
        { "Webhook ทำงานแล้ว", "Webhook works" },
        { "โหลดไม่สำเร็จ:", "failed to load:" },
        { "ส่งแจ้งเตือนตายไม่สำเร็จ:", "failed to send death alert:" },
        { "ใช้โหมดค้นหาสำรองแทน", "using fallback search instead" },
        { "executor นี้ไม่รองรับ setfpscap จึงล็อก FPS จริงไม่ได้", "this executor lacks setfpscap, so FPS cannot be truly locked" },
        { "executor ไม่มี request/http_request ที่ใช้งานได้", "executor has no usable request/http_request" },
        { "executor ไม่รองรับ firetouchinterest โหมด Pull จะเก็บของไม่ได้ ให้ใช้โหมด Walk", "executor lacks firetouchinterest; Pull mode cannot collect, use Walk mode" },
        { "การเก็บของ", "Collecting" },
        { "กำลังตรวจจับ... (กดปุ่มตรวจจับเพื่ออัปเดต)", "Detecting... (press the detect button to refresh)" },
        { "ขนาด GUI", "GUI size" },
        { "ขนาดตัวอักษร ESP", "ESP text size" },
        { "ขนาดตัวอักษร", "Text size" },
        { "ของในตัวตอนตาย", "Items on you at death" },
        { "ความเร็วดูด (โหมด Pull)", "Pull speed (Pull mode)" },
        { "คืนค่าเดิมได้", "reversible" },
        { "จำนวนไอเทมที่เห็น: ", "Items seen: " },
        { "จำนวนไอเทมสูงสุดต่อคน", "Max items per player" },
        { "จุดเริ่มต้น", "Start point" },
        { "ดันแถบสเตมิน่าให้เต็มตลอด วิ่ง/กระโดดได้เรื่อย ๆ", "Keeps the stamina bar full so you can sprint/jump endlessly" },
        { "ดูรายละเอียดเต็มใน Console (F9)", "See full details in Console (F9)" },
        { "ดูว่าสคริปต์เห็นไอเทมอะไรบ้าง (ผลออกที่ Console F9 ด้วย)", "Shows which items the script sees (also printed to Console F9)" },
        { "ดูไอเทมในตัวผู้เล่น", "View player items" },
        { "ตรวจจับระบบเกม", "Detect game systems" },
        { "ตรวจพบว่าตัวละครของคุณตาย/ถูกรีเซ็ต", "Your character died / was reset" },
        { "ตั้งค่า Recoil ของปืนเป็น 0 ที่ฝั่งเครื่องเรา ปิดแล้วคืนค่าเดิมให้", "Sets gun Recoil to 0 on your side; turning off restores the original" },
        { "ติ๊กเฉพาะระดับที่อยากเห็น (เช่น ติ๊กแค่ Epic, Legendary, Omega)", "Tick only the tiers you want to see (e.g. just Epic, Legendary, Omega)" },
        { "ทดสอบ Discord Webhook", "Test Discord Webhook" },
        { "ทดสอบ Item Management System", "Test Item Management System" },
        { "ทำเครื่องหมาย ▶ ที่ไอเทมที่กำลังถือ", "Mark the held item with ▶" },
        { "ทุกชิ้นในระยะพร้อมกัน", "all items in range at once" },
        { "ประสิทธิภาพ (เครื่องอ่อน)", "Performance (low-end devices)" },
        { "ปรับขนาดไม่ได้", "cannot resize" },
        { "ปิด UI และคืนค่าทุกอย่างกลับเป็นปกติ (ESP, กราฟิก, การดูด)", "Closes the UI and restores everything (ESP, graphics, vacuum)" },
        { "ปิดสคริปต์ทั้งหมด", "Unload script" },
        { "ปิดไว้ = สั่งเก็บทุกชิ้นพร้อมกันโดยไม่ย้ายของ (แนะนำ เพราะเกมคุมตำแหน่งของเอง)", "Off = trigger pickup for all items without moving them (recommended; the game controls item positions)" },
        { "ปืน + สเตมิน่า", "Gun + Stamina" },
        { "ปืนไม่มีแรงสะท้อน (จอไม่สั่น)", "No gun recoil (no screen shake)" },
        { "ปุ่มลัดเปิด/ปิด Auto-Collect", "Hotkey to toggle Auto-Collect" },
        { "ผู้เล่นใกล้ตัว (≤150 studs)", "Nearby players (≤150 studs)" },
        { "ไม่พบโมดูล Sprint ของเกม ฟีเจอร์นี้อาจใช้ไม่ได้ในแมพนี้", "Game Sprint module not found; this feature may not work on this map" },
        { "ไม่พบโฟลเดอร์", "Folder not found" },
        { "ไม่พบ (ใช้ค้นหาสำรอง)", "not found (using fallback search)" },
        { "ไม่พบ", "not found" },
        { "พบ", "found" },
        { "ยังไม่พบปืนในตัว ลองหยิบปืนมาไว้ในกระเป๋าแล้วสคริปต์จะจับให้เอง", "No gun on you yet. Put a gun in your backpack and the script will pick it up" },
        { "ยังไม่ได้ใส่ Webhook URL", "Webhook URL not set" },
        { "ย้ายของเข้าหาตัวด้วย (ทดลอง)", "Also move items toward you (experimental)" },
        { "รวมเป็นชุด ไม่แจ้งทีละชิ้นให้จอรก", "Batched, no per-item popups" },
        { "ระยะตรวจจับ", "Detection range" },
        { "ระยะแสดงผล", "Display range" },
        { "ลูปเดียว", "single loop" },
        { "ล็อก FPS ที่", "Lock FPS at" },
        { "ล็อก FPS", "FPS Lock" },
        { "สคริปต์", "Script" },
        { "สถานะระบบเกม", "Game system status" },
        { "สถานะ", "Status" },
        { "สีกำหนดเอง (ใช้เมื่อเลือก Custom)", "Custom color (used when Custom is selected)" },
        { "สีชื่อผู้เล่นตามไอเทมที่แรร์สุดในตัว", "Color player name by their rarest item" },
        { "สีป้าย ESP", "ESP label color" },
        { "สเตมิน่าไม่จำกัด", "Infinite stamina" },
        { "สเตมิน่า", "Stamina" },
        { "ส่ง Death Webhook ไม่สำเร็จ: ", "Death webhook failed: " },
        { "ส่งข้อความทดสอบ 1 ครั้ง", "Send 1 test message" },
        { "ส่งสาเหตุ ตำแหน่ง ผู้เล่นใกล้ตัว และของในตัวตอนตาย ไป Discord", "Sends cause, position, nearby players and items on you at death to Discord" },
        { "ส่งสำเร็จ ดูในห้อง Discord ได้เลย", "Sent! Check your Discord channel" },
        { "ส่งไม่สำเร็จ: ", "Send failed: " },
        { "หน่วย % (ค่าเริ่มต้น 100)", "Unit % (default 100)" },
        { "หน่วย studs (ค่าเริ่มต้น 20)", "Unit studs (default 20)" },
        { "หน่วย studs (สูงสุด 5,000)", "Unit studs (max 5,000)" },
        { "หน้าตา ESP", "ESP style" },
        { "หน้าตา", "Style" },
        { "หาหน้าต่าง Fluent ไม่เจอ", "Fluent window not found" },
        { "เก็บสำเร็จแล้ว: ", "Collected: " },
        { "เก็บสำเร็จ ", "Collected " },
        { "เก็บสำเร็จ", "Collected" },
        { "เครื่องมือ", "Tools" },
        { "เช็กว่าเกม/แมพนี้มีระบบสเตมิน่าและแรงสะท้อนที่สคริปต์รองรับหรือไม่", "Checks whether this game/map has stamina and recoil systems the script supports" },
        { "เปิด/ปิด Auto-Collect", "Toggle Auto-Collect" },
        { "เปิด/ปิด ESP ไอเทมที่ตกพื้น", "Toggle dropped-item ESP" },
        { "เปิด/ปิด Player ESP", "Toggle Player ESP" },
        { "เลือกเพดาน FPS ตั้งแต่ 15-240", "Pick an FPS cap from 15-240" },
        { "แจ้งเตือนเมื่อเก็บสำเร็จ", "Notify on successful collect" },
        { "แจ้งเตือนเมื่อเราตาย", "Alert when I die" },
        { "แรร์สุดในชุดนี้: ", "Rarest in this batch: " },
        { "แสดง ชื่อ | ความแรร์ | ระยะ เหนือไอเทม", "Shows name | rarity | distance above each item" },
        { "แสดงของแรร์สุดก่อน ที่เกินจะย่อเป็น +N", "Rarest first; overflow collapses into +N" },
        { "แสดงชื่อผู้เล่น + ระยะ", "Show player name + distance" },
        { "แสดงชื่อไอเทมของผู้เล่นคนอื่น สีตามความแรร์ (ทอง = Legendary, ม่วง = Epic)", "Shows other players' item names colored by rarity (gold = Legendary, purple = Epic)" },
        { "แสดงเฉพาะความแรร์", "Show only rarities" },
        { "โฟลเดอร์: ", "Folder: " },
        { "โหมด Auto-Collect", "Auto-Collect mode" },
        { "โหมดลดกราฟิก (เพิ่ม FPS)", "Low-graphics mode (more FPS)" },
        { "โหลดสำเร็จ กด RightControl เพื่อซ่อน/แสดงหน้าต่าง", "Loaded. Press RightControl to hide/show the window" },
        { "ใกล้สุด: ", "Nearest: " },
        { "ใช้ setfpscap เพื่อกำหนดเพดาน FPS ของ executor", "Uses setfpscap to cap the executor FPS" },
        { "ใช้ไม่ได้", "unavailable" },
        { "ใส่ webhook ของเซิร์ฟเวอร์ Discord ที่คุณเป็นเจ้าของ", "Enter a webhook for a Discord server you own" },
        { "ไม่มีแรงสะท้อน", "No recoil" },
        { "ไม่มี", "None" },
        { "ไม่เจอ workspace[", "Could not find workspace[" },
        { "☠ ผู้เล่นตาย", "☠ Player died" },
        { "มุดดิน (Burrow)", "Burrow (go underground)" },
        { "คำเตือน", "Warning" },
        { "ตำแหน่งตัวละครจะอยู่ใต้พื้นจริง เซิร์ฟเวอร์เห็นตำแหน่งนี้ เกมนี้มีระบบตรวจจับ อาจโดนเตะ/แบนได้ ใช้ด้วยความเสี่ยงของตัวเอง", "Your character really sits below the ground and the server sees that position. This game has detection, so you may be kicked/banned. Use at your own risk" },
        { "เปิดระบบมุดดิน", "Enable burrow" },
        { "เปิดแล้ว: คอมกดปุ่มที่ตั้งไว้ (ค่าเริ่มต้น Z) / มือถือกดปุ่ม Snap เพื่อมุดลง กดซ้ำเพื่อขึ้น", "When on: press your key (default Z) on PC / tap the Snap button on mobile to sink; press again to come up" },
        { "ปุ่มมุดดิน (คอม)", "Burrow key (PC)" },
        { "ระยะที่มุดลง", "Burrow depth" },
        { "หน่วย studs (ค่าเริ่มต้น 10)", "Unit studs (default 10)" },
        { "แสดงปุ่ม Snap บนหน้าจอ", "Show on-screen Snap button" },
        { "สำหรับมือถือ ลากย้ายตำแหน่งปุ่มได้", "For mobile; you can drag the button around" },
        { "Snap ตอนนี้", "Snap now" },
        { "ทดสอบมุดลง/ขึ้นจากเมนู", "Sink/rise from the menu" },
        { "มุดดิน", "Burrow" },
        { "มุดลงแล้ว ลึก", "Sunk, depth" },
        { "กดอีกครั้งเพื่อขึ้น", "press again to rise" },
        { "ขึ้นมาแล้ว", "Back up" },
        { "ตัวละครยังไม่พร้อม", "Character not ready" },
        { "ภาษา / Language", "Language / ภาษา" },
        { "เปลี่ยนภาษาเรียบร้อย", "Language changed" },
        { "จำค่าอัตโนมัติ", "Auto remember settings" },
        { "จำค่าที่ตั้งไว้อัตโนมัติ", "Remember settings automatically" },
        { "บันทึกทุกครั้งที่เปลี่ยนค่า เข้าเกม/แมพใหม่แล้วจะโหลดค่าเดิมให้เอง (เก็บในไฟล์ของเครื่องนี้ รวมถึง Webhook URL อย่าแชร์ไฟล์)", "Saves on every change and reloads it next time you join (stored in a file on this device, including your Webhook URL, so do not share the file)" },
        { "บันทึกตอนนี้", "Save now" },
        { "ล้างค่าที่จำไว้", "Clear remembered settings" },
        { "ลบไฟล์ที่จำไว้ ครั้งหน้าจะเริ่มจากค่าเริ่มต้น", "Deletes the saved file; next run starts from defaults" },
        { "บันทึกค่าแล้ว", "Settings saved" },
        { "โหลดค่าที่เคยตั้งไว้แล้ว", "Loaded your previous settings" },
        { "ลบค่าที่จำไว้แล้ว", "Remembered settings deleted" },
        { "บันทึกไม่สำเร็จ", "Save failed" },
        { "กรุณาเปิดระบบมุดดินก่อน", "Enable burrow first" },
        { "ภาษา (Language)", "Language" },
        { "ปุ่มเปิด/ปิดเมนู", "Menu toggle button" },
        { "แสดงปุ่มเมนูลอยบนหน้าจอ", "Show floating menu button" },
        { "กดเพื่อซ่อน/แสดงหน้าต่างสคริปต์ ใช้บนมือถือได้ (ไม่ต้องใช้ RightControl) ลากย้ายตำแหน่งได้", "Tap to hide/show the script window. Works on mobile (no RightControl needed); drag to move it" },
        { "เลือกอุปกรณ์ของคุณ", "Choose your device" },
        { "โทรศัพท์", "Phone" },
        { "คอม", "PC" },
        { "เลือกแล้วจะจำไว้ เปลี่ยนภายหลังได้ที่แท็บ General", "Your choice is remembered; change it later in the General tab" },
        { "อุปกรณ์ (Device)", "Device" },
        { "เปลี่ยนอุปกรณ์แล้ว ขนาดหน้าต่างจะปรับเมื่อรันสคริปต์ครั้งหน้า", "Device changed; window size updates next time you run the script" },
        { "ปลาที่ตกได้ (Fish Log)", "Caught fish (Fish Log)" },
        { "เปิดตัวติดตามปลา", "Enable fish tracker" },
        { "ดูว่าปลาที่ตกได้เป็นระดับไหน นับสถิติ และเตือนเมื่อมีปลาที่ไม่ใช่ระดับที่เก็บสะสมในกระเป๋า (ไม่ได้ทิ้งของให้ ใช้ Bulk Drop ของเกมเอง)", "Shows the rarity of each fish you catch, keeps stats, and warns when fish outside your keep list pile up (it does not drop anything; use the game's Bulk Drop)" },
        { "ระดับปลาที่ต้องการเก็บ", "Fish rarities to keep" },
        { "ติ๊กระดับที่ต้องการเก็บ ที่เหลือจะถูกนับเป็นปลาที่ควรทิ้ง", "Tick the rarities to keep; the rest are counted as fish to drop" },
        { "แจ้งเตือนทุกครั้งที่ตกได้", "Notify on every catch" },
        { "ส่ง Discord เมื่อได้ปลาระดับที่เก็บ / กระเป๋ามีปลาควรทิ้งเยอะ", "Send to Discord when a keep-rarity fish is caught / too many drop-fish" },
        { "ต้องตั้ง Webhook URL ในแท็บ Discord ก่อน", "Set the Webhook URL in the Discord tab first" },
        { "เตือนเมื่อปลาที่ควรทิ้งสะสมถึง", "Warn when drop-fish reach" },
        { "หน่วย ตัว", "Unit: fish" },
        { "สแกนกระเป๋าตอนนี้", "Scan backpack now" },
        { "แสดงรายการปลาในกระเป๋าใน Console (F9)", "Prints the fish in your backpack to Console (F9)" },
        { "รีเซ็ตตัวนับ", "Reset counters" },
        { "ได้ปลา", "Caught fish" },
        { "เก็บไว้", "keep" },
        { "ควรทิ้ง", "drop" },
        { "ปลาที่ควรทิ้งสะสมถึง", "Drop-fish reached" },
        { "ใช้ Bulk Drop ของเกมทิ้งทีเดียวได้เลย", "you can use the game's Bulk Drop to drop them at once" },
        { "จับได้รอบนี้", "Caught this session" },
        { "ในกระเป๋าที่ควรทิ้ง", "In backpack to drop" },
        { "ยังไม่มีปลา", "No fish yet" },
        { "สรุปปลาในกระเป๋า", "Backpack fish summary" },
        { "ตัว", "fish" },
        { "ได้ปลาระดับที่เก็บ", "Caught a keep-rarity fish" },
        { "โหลดค่าจาก Configs แล้ว", "Loaded values from Configs" }
    }

    local function escPattern(text)
        return (text:gsub("[%^%$%(%)%%%.%[%]%*%+%-%?]", "%%%0"))
    end
    local function escReplace(text)
        return (text:gsub("%%", "%%%%"))
    end

    local compiled = {}
    for _, p in ipairs(PAIRS) do
        table.insert(compiled, { pat = escPattern(p[1]), rep = escReplace(p[2]), len = #p[1] })
    end
    -- แทนวลียาวก่อน กันวลีสั้นไปแทนกลางประโยค
    table.sort(compiled, function(a, b) return a.len > b.len end)

    local function hasThai(text)
        return text:find("\224[\184\185]") ~= nil
    end
    Ext.hasThai = hasThai

    Ext.lang = "th"
    pcall(function()
        if isfile and isfile(LANG_FILE) then
            local v = tostring(readfile(LANG_FILE)):gsub("%s", "")
            if v == "en" or v == "th" then Ext.lang = v end
        end
    end)

    function Ext.saveLang()
        pcall(function()
            if makefolder and isfolder and not isfolder("ItemManagementSystem") then
                makefolder("ItemManagementSystem")
            end
            if writefile then writefile(LANG_FILE, Ext.lang) end
        end)
    end

    T = function(text)
        if Ext.lang ~= "en" or type(text) ~= "string" or not hasThai(text) then return text end
        for _, c in ipairs(compiled) do
            text = text:gsub(c.pat, c.rep)
        end
        return text
    end
end

--// ============================== Discord Webhook ==============================
local function getRequestFunction()
    local g = (getgenv and getgenv()) or _G or {}
    return (syn and syn.request)
        or (http and http.request)
        or (fluxus and fluxus.request)
        or http_request
        or request
        or g.request
        or g.http_request
end

local function validWebhookUrl(url)
    return url:match("^https://[%w%.%-]*discord%.com/api/webhooks/%d+/[%w%-_]+") ~= nil
        or url:match("^https://[%w%.%-]*discordapp%.com/api/webhooks/%d+/[%w%-_]+") ~= nil
end

-- ส่งข้อความเข้า Discord  คืนค่า true หรือ false + ข้อความผิดพลาด
local function sendDiscordWebhook(title, content, fields, color)
    local url = tostring(CONFIG.DISCORD_WEBHOOK_URL or ""):gsub("%s+", "")
    if url == "" then return false, "ยังไม่ได้ใส่ Webhook URL" end
    if not validWebhookUrl(url) then
        return false, "URL ไม่ถูกต้อง (ต้องขึ้นต้นด้วย https://discord.com/api/webhooks/...)"
    end

    local req = getRequestFunction()
    if type(req) ~= "function" then
        return false, "executor ไม่มี request/http_request ที่ใช้งานได้"
    end

    local translatedFields = {}
    for i, f in ipairs(fields or {}) do
        translatedFields[i] = { name = T(tostring(f.name)), value = T(tostring(f.value)), inline = f.inline }
    end

    local payload = {
        username = "Item Management System",
        embeds = {{
            title = T(tostring(title)),
            description = T(tostring(content)),
            color = color or 15022389,
            fields = translatedFields,
            timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        }},
    }

    -- แท็ก Discord User ID (ถ้ามี)
    local pingId = tostring(CONFIG.DISCORD_PING_ID or ""):gsub("%D", "")
    if pingId ~= "" then
        payload.content = "<@" .. pingId .. ">"
        payload.allowed_mentions = { users = { pingId } }
    else
        payload.allowed_mentions = { parse = {} }
    end

    local okEncode, body = pcall(function()
        return HttpService:JSONEncode(payload)
    end)
    if not okEncode then return false, tostring(body) end

    local okReq, res = pcall(function()
        return req({
            Url = url,
            Method = "POST",
            Headers = { ["Content-Type"] = "application/json" },
            Body = body,
        })
    end)
    if not okReq then return false, tostring(res) end

    if type(res) == "table" then
        local code = tonumber(res.StatusCode or res.Status or res.status_code)
        if code and code >= 200 and code < 300 then
            return true, res
        end
        local hint = ""
        if code == 404 then hint = " (Webhook ถูกลบหรือ URL ผิด)"
        elseif code == 429 then hint = " (ส่งถี่เกิน รอสักครู่)"
        elseif code == 401 or code == 403 then hint = " (Webhook ไม่ถูกต้อง)" end
        return false, "HTTP " .. tostring(code or "?") .. hint .. " " .. tostring(res.Body or res.body or "")
    end

    return res and true or false, res
end

--// ============================== โหลดไลบรารี Fluent ==============================
local function loadLib(url)
    local ok, result = pcall(function()
        return loadstring(game:HttpGet(url))()
    end)
    if ok then return result end
    warn("[ItemSystem] โหลดไม่สำเร็จ:", url, result)
    return nil
end

local Fluent = loadLib("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua")
if not Fluent then return end -- ไม่มี UI หลักก็ทำงานต่อไม่ได้

-- Addon เป็นตัวเสริม โหลดไม่ได้ก็ยังใช้สคริปต์ได้
local SaveManager      = loadLib("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua")
local InterfaceManager = loadLib("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua")

-- เปลี่ยนภาษาของหน้าต่างที่สร้างไว้แล้ว (เดินดู TextLabel/TextButton ทั้งหมดของ Fluent)
do
    local Orig = setmetatable({}, { __mode = "k" })  -- label -> { ข้อความไทยเดิม, ข้อความที่แปลแล้ว }

    function Ext.apply()
        local gui = Fluent.GUI
        if typeof(gui) ~= "Instance" then return end
        for _, d in ipairs(gui:GetDescendants()) do
            if d:IsA("TextLabel") or d:IsA("TextButton") then
                local cur = d.Text
                local rec = Orig[d]
                if Ext.lang == "en" then
                    if Ext.hasThai(cur) then
                        local tr = T(cur)
                        if tr ~= cur then
                            Orig[d] = { cur, tr }
                            d.Text = tr
                        end
                    end
                elseif rec and cur == rec[2] then
                    d.Text = rec[1]
                    Orig[d] = nil
                end
            end
        end
    end

    function Ext.setLang(code)
        if code ~= "en" and code ~= "th" then return end
        Ext.lang = code
        Ext.saveLang()
        pcall(Ext.apply)
    end
end

local function notify(title, content, duration)
    pcall(function()
        Fluent:Notify({ Title = T(title), Content = T(content), Duration = duration or 4 })
    end)
end

-- ========== ตัวช่วยปุ่มลอย (กดครั้งเดียวติด / ลากได้ / ใช้ได้ทั้งมือถือและคอม) ==========
do
    local UIS = game:GetService("UserInputService")

    -- สร้าง ScreenGui ในที่ที่เสถียรที่สุดที่ executor รองรับ (gethui > CoreGui > PlayerGui)
    function Ext.makeGui(name, order)
        local gui = Instance.new("ScreenGui")
        gui.Name = name
        gui.ResetOnSpawn = false
        gui.IgnoreGuiInset = true
        gui.DisplayOrder = order or 50
        pcall(function() gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling end)
        pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)

        local parent
        pcall(function() if gethui then parent = gethui() end end)
        if not parent then pcall(function() parent = game:GetService("CoreGui") end) end
        if parent then pcall(function() gui.Parent = parent end) end
        if not gui.Parent then gui.Parent = LocalPlayer:WaitForChild("PlayerGui") end
        return gui
    end

    -- ทำให้ปุ่ม "แตะ = เรียก onTap" และ "ลาก = ย้ายที่" โดยไม่สับสนกัน
    -- ติดตามเฉพาะนิ้ว/เมาส์ที่แตะปุ่มนี้ (นิ้วอื่น เช่น จอยเดิน/หมุนกล้อง ไม่ทำให้การแตะพลาดอีก)
    function Ext.makeTappable(btn, onTap)
        local active, moved, startInput, startAbs

        btn.InputBegan:Connect(function(input)
            local t = input.UserInputType
            if t == Enum.UserInputType.Touch or t == Enum.UserInputType.MouseButton1 then
                active = input
                moved = false
                startInput = input.Position
                startAbs = btn.AbsolutePosition
            end
        end)

        -- [FIXED] UIS เป็น service ระดับเกม; เก็บ listener ไว้ตัดเมื่อปุ่มถูกทำลาย
        local changedConnection = UIS.InputChanged:Connect(function(input)
            if not active or not btn.Parent then return end
            local isTouch = active.UserInputType == Enum.UserInputType.Touch
            local mine = (isTouch and input == active)
                or (not isTouch and input.UserInputType == Enum.UserInputType.MouseMovement)
            if not mine then return end

            local delta = Vector2.new(input.Position.X - startInput.X, input.Position.Y - startInput.Y)
            if delta.Magnitude > 10 then moved = true end
            if moved and btn.Parent then
                local parentSize = btn.Parent.AbsoluteSize
                local x = math.clamp(startAbs.X + delta.X, 0, math.max(0, parentSize.X - btn.AbsoluteSize.X))
                local y = math.clamp(startAbs.Y + delta.Y, 0, math.max(0, parentSize.Y - btn.AbsoluteSize.Y))
                btn.Position = UDim2.fromOffset(x, y)
            end
        end)

        local endedConnection = UIS.InputEnded:Connect(function(input)
            if not active or not btn.Parent then active = nil; return end
            local isTouch = active.UserInputType == Enum.UserInputType.Touch
            local ended = (isTouch and input == active)
                or (not isTouch and input.UserInputType == Enum.UserInputType.MouseButton1)
            if not ended then return end
            local wasMoved = moved
            active = nil
            if not wasMoved then
                local ok, err = pcall(onTap)
                if not ok then warn("[ItemSystem] tap error:", err) end
            end
        end)
        btn.Destroying:Connect(function()
            active = nil
            if changedConnection then pcall(function() changedConnection:Disconnect() end) end
            if endedConnection then pcall(function() endedConnection:Disconnect() end) end
        end)
    end
end

-- ========== เลือกอุปกรณ์ (โทรศัพท์ / คอม) ==========
-- ถามครั้งแรกที่รัน แล้วจำไว้ในไฟล์ เปลี่ยนภายหลังได้ที่แท็บ General
do
    local PLATFORM_FILE = "ItemManagementSystem/platform.txt"
    local UIS = game:GetService("UserInputService")

    function Ext.savePlatform()
        pcall(function()
            if makefolder and isfolder and not isfolder("ItemManagementSystem") then
                makefolder("ItemManagementSystem")
            end
            if writefile then writefile(PLATFORM_FILE, Ext.platform) end
        end)
    end

    local saved
    pcall(function()
        if isfile and isfile(PLATFORM_FILE) then
            local v = tostring(readfile(PLATFORM_FILE)):gsub("%s", "")
            if v == "mobile" or v == "pc" then saved = v end
        end
    end)

    if saved then
        Ext.platform = saved
    else
        local auto = (UIS.TouchEnabled and not UIS.KeyboardEnabled) and "mobile" or "pc"
        local chosen
        local okPick = pcall(function()
            local gui = Ext.makeGui("IMS_DevicePicker", 200)

            local frame = Instance.new("Frame")
            frame.AnchorPoint = Vector2.new(0.5, 0.5)
            frame.Position = UDim2.fromScale(0.5, 0.5)
            frame.Size = UDim2.fromOffset(300, 190)
            frame.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
            frame.Parent = gui
            local fc = Instance.new("UICorner")
            fc.CornerRadius = UDim.new(0, 12)
            fc.Parent = frame
            local fs = Instance.new("UIStroke")
            fs.Color = Color3.fromRGB(120, 170, 255)
            fs.Thickness = 2
            fs.Parent = frame

            local title = Instance.new("TextLabel")
            title.BackgroundTransparency = 1
            title.Size = UDim2.new(1, 0, 0, 44)
            title.Font = Enum.Font.GothamBold
            title.TextSize = 18
            title.TextColor3 = Color3.new(1, 1, 1)
            title.Text = T("เลือกอุปกรณ์ของคุณ")
            title.Parent = frame

            local function makeChoice(text, value, x)
                local b = Instance.new("TextButton")
                b.Size = UDim2.new(0.5, -20, 0, 64)
                b.Position = UDim2.new(x, x == 0 and 14 or 6, 0, 52)
                b.BackgroundColor3 = (value == auto) and Color3.fromRGB(60, 110, 200) or Color3.fromRGB(44, 44, 54)
                b.TextColor3 = Color3.new(1, 1, 1)
                b.Font = Enum.Font.GothamBold
                b.TextSize = 17
                b.Text = text
                b.Parent = frame
                local c = Instance.new("UICorner")
                c.CornerRadius = UDim.new(0, 10)
                c.Parent = b
                b.Activated:Connect(function() chosen = value end)
            end
            makeChoice(T("โทรศัพท์"), "mobile", 0)
            makeChoice(T("คอม"), "pc", 0.5)

            local hint = Instance.new("TextLabel")
            hint.BackgroundTransparency = 1
            hint.Position = UDim2.new(0, 10, 0, 128)
            hint.Size = UDim2.new(1, -20, 0, 50)
            hint.Font = Enum.Font.Gotham
            hint.TextSize = 13
            hint.TextWrapped = true
            hint.TextColor3 = Color3.fromRGB(170, 170, 185)
            hint.Text = T("เลือกแล้วจะจำไว้ เปลี่ยนภายหลังได้ที่แท็บ General")
            hint.Parent = frame

            -- ไม่เลือกภายใน 45 วินาที ใช้ค่าที่เดาจากอุปกรณ์
            local waited = 0
            while not chosen and waited < 45 do
                task.wait(0.1)
                waited = waited + 0.1
            end
            gui:Destroy()
        end)
        Ext.platform = chosen or auto
        Ext.savePlatform()
        if not okPick then warn("[ItemSystem] เปิดหน้าเลือกอุปกรณ์ไม่สำเร็จ ใช้ค่าอัตโนมัติ:", Ext.platform) end
    end
end


--// ============================== ข้อมูลความแรร์ ==============================
-- เรียงจากแรร์สุดไปน้อยสุด (Uncommon ต้องมาก่อน Common เพราะชื่อซ้อนกัน)
local RARITY_LIST = {
    "Divine", "Secret", "Omega", "Exotic", "Mythic",
    "Legendary", "Epic", "Rare", "Uncommon", "Common",
}

local RARITY_COLORS = {
    Common    = Color3.fromRGB(200, 200, 200),
    Uncommon  = Color3.fromRGB(90, 220, 90),
    Rare      = Color3.fromRGB(70, 140, 255),
    Epic      = Color3.fromRGB(180, 80, 255),
    Legendary = Color3.fromRGB(255, 170, 0),
    Mythic    = Color3.fromRGB(255, 60, 90),
    Omega     = Color3.fromRGB(255, 0, 200),
    Exotic    = Color3.fromRGB(0, 230, 200),
    Secret    = Color3.fromRGB(120, 120, 255),
    Divine    = Color3.fromRGB(255, 255, 120),
}

-- สีตามความแรร์ของเกม (Legendary = ทอง, Epic = ม่วง, Omega = แดง)
RARITY_COLORS.Common    = Color3.fromRGB(255, 255, 255)
RARITY_COLORS.Uncommon  = Color3.fromRGB(99, 255, 52)
RARITY_COLORS.Rare      = Color3.fromRGB(51, 170, 255)
RARITY_COLORS.Epic      = Color3.fromRGB(237, 44, 255)
RARITY_COLORS.Legendary = Color3.fromRGB(255, 170, 0)
RARITY_COLORS.Omega     = Color3.fromRGB(255, 20, 51)

-- ลำดับความแรร์ (เลขน้อย = แรร์กว่า) ใช้สรุปแจ้งเตือน
local RARITY_RANK = {}
for i, r in ipairs(RARITY_LIST) do RARITY_RANK[r] = i end

-- [FIXED v16] รุ่นนี้เน้น Main / Player ESP / Combat / Aimbot / Movement / General / Settings
-- Auto Farm, Fishing Farm และเส้นทางรถถูกนำออกแล้ว; ไม่เปิดงานฟาร์มอัตโนมัติอยู่เบื้องหลัง
--// ============================== ตัวแปรสถานะ ==============================
local State = {
    CollectEnabled = false,
    ESPEnabled     = false,
    Mode           = "Pull",   -- "Pull" หรือ "Walk"
    Range          = 20,       -- ระยะตรวจจับ (studs)
    PullSpeed      = 1,        -- 1 = ดูดทันที, น้อยกว่านี้ = ค่อย ๆ ดูด
    NotifyCollect  = true,
    ESPColorMode   = "Rarity", -- "Rarity" หรือ "Custom"
    ESPColor       = Color3.fromRGB(255, 255, 255),
    ESPTextSize    = 14,
    Home           = nil,      -- จุดเริ่มต้นก่อนเดินไปเก็บ (โหมด Walk)
    -- Player ESP (ดูไอเทมในตัวผู้เล่นอื่น)
    PlayerESPEnabled = false,
    PERange        = 5000,     -- ระยะแสดงผล (studs)
    PEShowName     = true,     -- แสดงชื่อผู้เล่น + ระยะ
    NoRecoil       = false,    -- ปืนไม่มีแรงสะท้อน
    InfStamina     = false,    -- สเตมิน่าเต็มตลอด
    PEHeaderByRarity = true,   -- สีชื่อผู้เล่นตามไอเทมที่แรร์ที่สุด
    PEShowHeld     = true,     -- ทำเครื่องหมายไอเทมที่กำลังถืออยู่
    PEMaxLines     = 10,       -- จำนวนบรรทัดไอเทมสูงสุดต่อคน
    PETextSize     = 12,
    PERarityFilter = { common = true, uncommon = true, rare = true, epic = true, legendary = true, omega = true },
    PullMove       = false,    -- false = สั่งเก็บทุกชิ้นพร้อมกันโดยไม่ย้ายของ (เกมคุมตำแหน่งของเอง)

    -- Discord
    DeathWebhook   = true,
}

local Stats       = {
    Count = 0,
    PlayerItemCache = setmetatable({}, { __mode = "k" }),
    ZoneCache = setmetatable({}, { __mode = "k" }),
    AnchorCache = setmetatable({}, { __mode = "k" }),
    ItemInfoCache = setmetatable({}, { __mode = "k" }),
    FpsFrames = 0,
    FpsConn = nil,
    VacLast = 0,
}
local Counted     = setmetatable({}, { __mode = "k" }) -- กันนับซ้ำ
local ESPs        = {}                                  -- [item] = { Gui, Label, Conn }
local Skipped     = setmetatable({}, { __mode = "k" })  -- [item] = เวลาที่จะลองใหม่ (Walk)
local Walking     = false
local WarnedNoFolder = false
local FallbackCache  = { Time = 0, Items = {} }

-- UI objects ที่ต้องอัปเดตจากโค้ดภายหลัง
local StatusPara, FpsPara, FpsLockPara, Window

--// ============================== FPS Lock ==============================
-- ล็อก FPS ผ่าน setfpscap ของ executor (ถ้ามี)
local FpsLock = {
    Enabled = false,
    Cap = 60,
    Supported = type(setfpscap) == "function",
    LastNotice = false,
}

local function setFpsLock(enabled, cap)
    FpsLock.Enabled = enabled and true or false
    FpsLock.Cap = math.clamp(math.floor(tonumber(cap) or 60), 15, 240)

    local setter = setfpscap
    if type(setter) ~= "function" then
        FpsLock.Supported = false
        if not FpsLock.LastNotice then
            FpsLock.LastNotice = true
            notify("FPS Lock", "executor นี้ไม่รองรับ setfpscap จึงล็อก FPS จริงไม่ได้", 6)
        end
        if FpsLockPara then
            setParagraph(FpsLockPara, "FPS Lock: ไม่รองรับใน executor นี้")
        end
        return false
    end

    local target = FpsLock.Enabled and FpsLock.Cap or 0
    local ok, err = pcall(setter, target)
    if not ok then
        FpsLock.Supported = false
        if FpsLockPara then
            setParagraph(FpsLockPara, "FPS Lock: ตั้งค่าไม่สำเร็จ")
        end
        warn("[ItemSystem] setfpscap error:", err)
        return false
    end

    FpsLock.Supported = true
    FpsLock.LastNotice = false
    if FpsLockPara then
        setParagraph(FpsLockPara, FpsLock.Enabled
            and ("FPS Lock: " .. tostring(FpsLock.Cap) .. " FPS")
            or "FPS Lock: ปิด")
    end
    return true
end


--// ============================== ฟังก์ชันช่วยเหลือ ==============================

local function normalize(name)
    return (string.lower(tostring(name)):gsub("[%s_%-]", ""))
end

local function getRoot()
    local char = LocalPlayer.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
    local char = LocalPlayer.Character
    return char and char:FindFirstChildOfClass("Humanoid")
end

-- อัปเดตข้อความของ Paragraph (Fluent เวอร์ชันต่าง ๆ ใช้ชื่อเมธอดไม่เหมือนกัน จึงลองหลายแบบ)
local function setParagraph(para, text)
    if not para then return end
    text = T(text)
    local ok = pcall(function() para:SetDesc(text) end)
    if not ok then pcall(function() para:SetValue(text) end) end
end

local function findDropFolder()
    local exact = Workspace:FindFirstChild(CONFIG.DROP_FOLDER_NAME)
    if exact then return exact end

    local target = normalize(CONFIG.DROP_FOLDER_NAME)
    for _, child in ipairs(Workspace:GetChildren()) do
        if normalize(child.Name) == target then return child end
    end
    return nil
end

-- ไม่ใช่ตัวละคร/NPC (กันดูดตัวคน)
local function isNotCharacter(inst)
    if inst == LocalPlayer.Character then return false end
    if inst:IsA("Model") then
        if inst:FindFirstChildOfClass("Humanoid") then return false end
        if Players:GetPlayerFromCharacter(inst) then return false end
    end
    return true
end

local function looksLikeZone(inst)
    if not inst:IsA("BasePart") then return false end
    local n = normalize(inst.Name)
    return n == normalize(CONFIG.ZONE_NAME) or string.find(n, "pickup", 1, true) ~= nil
end

local function getZone(item)
    if looksLikeZone(item) then return item end

    local cached = Stats.ZoneCache[item]
    if cached and cached.Parent then
        return cached
    end

    local exact = item:FindFirstChild(CONFIG.ZONE_NAME, true)
    if exact and exact:IsA("BasePart") then
        Stats.ZoneCache[item] = exact
        return exact
    end
    for _, d in ipairs(item:GetDescendants()) do
        if looksLikeZone(d) then
            Stats.ZoneCache[item] = d
            return d
        end
    end
    return nil
end

local function getAnchorPart(item)
    local cached = Stats.AnchorCache[item]
    if cached and cached.Parent then
        return cached
    end

    local zone = getZone(item)
    if zone then
        Stats.AnchorCache[item] = zone
        return zone
    end
    if item:IsA("BasePart") then
        Stats.AnchorCache[item] = item
        return item
    end
    if item:IsA("Tool") then
        local h = item:FindFirstChild("Handle")
        if h and h:IsA("BasePart") then
            Stats.AnchorCache[item] = h
            return h
        end
    end
    local found = item:FindFirstChildWhichIsA("BasePart", true)
    if found then Stats.AnchorCache[item] = found end
    return found
end

local function getItemPosition(item)
    local part = getAnchorPart(item)
    if part then return part.Position end
    if item:IsA("Model") then return item:GetPivot().Position end
    return nil
end

-- ชิ้นที่ย้ายตำแหน่งได้ (Model/Part ย้ายได้เลย, Tool ย้าย Handle)
local function getMovable(item)
    if item:IsA("PVInstance") then return item end
    return getAnchorPart(item)
end

local function readInfo(item, names)
    local zone = getZone(item)
    for _, key in ipairs(names) do
        local v = item:GetAttribute(key)
        if v == nil and zone then v = zone:GetAttribute(key) end
        if v ~= nil then return tostring(v) end

        local obj = item:FindFirstChild(key, true)
        if obj and obj:IsA("ValueBase") then return tostring(obj.Value) end
    end
    return nil
end

local function getItemName(item)
    local cached = Stats.ItemInfoCache[item]
    if cached and cached.Name then return cached.Name end

    local name = readInfo(item, { "ItemName", "DisplayName", "Name" }) or item.Name
    name = name:gsub("%s*[Mm][Oo][Dd][Ee][Ll]$", "")
    if name == "" then name = item.Name end

    Stats.ItemInfoCache[item] = Stats.ItemInfoCache[item] or {}
    Stats.ItemInfoCache[item].Name = name
    return name
end

-- ความแรร์: Attribute/Value -> คำในชื่อ -> Common
local function getRarity(item)
    local cached = Stats.ItemInfoCache[item]
    if cached and cached.Rarity then return cached.Rarity end

    local r = readInfo(item, { "Rarity", "Tier", "Grade" })
    if not r then
        local lowerName = string.lower(item.Name)
        for _, rarity in ipairs(RARITY_LIST) do
            if string.find(lowerName, string.lower(rarity), 1, true) then
                r = rarity
                break
            end
        end
    end
    r = r or "Common"

    Stats.ItemInfoCache[item] = Stats.ItemInfoCache[item] or {}
    Stats.ItemInfoCache[item].Rarity = r
    return r
end

--// ============================== รวบรวมไอเทมทั้งหมด ==============================

local function fallbackItems()
    if os.clock() - FallbackCache.Time < CONFIG.FALLBACK_REFRESH then
        return FallbackCache.Items
    end

    local items, seen = {}, {}
    for _, d in ipairs(Workspace:GetDescendants()) do
        if looksLikeZone(d) then
            local item = d:FindFirstAncestorWhichIsA("Model") or d
            if not seen[item] and isNotCharacter(item) then
                seen[item] = true
                table.insert(items, item)
            end
        end
    end

    FallbackCache.Time = os.clock()
    FallbackCache.Items = items
    return items
end

local function getAllItems()
    local folder = findDropFolder()
    if not folder then
        if not WarnedNoFolder then
            WarnedNoFolder = true
            notify("ไม่พบโฟลเดอร์", "ไม่เจอ workspace[\"" .. CONFIG.DROP_FOLDER_NAME .. "\"] ใช้โหมดค้นหาสำรองแทน", 5)
        end
        return fallbackItems()
    end

    local items = {}
    for _, child in ipairs(folder:GetChildren()) do
        if child:IsA("Folder") then
            for _, sub in ipairs(child:GetChildren()) do
                if isNotCharacter(sub) then table.insert(items, sub) end
            end
        elseif isNotCharacter(child) then
            table.insert(items, child)
        end
    end

    -- เมื่อเจอโฟลเดอร์หลักแล้วไม่สแกนทั้ง Workspace ซ้ำทุกครั้ง
    -- fallback จะใช้เฉพาะกรณีไม่พบ drop item จริง ๆ เพื่อลดภาระเครื่องในเซิร์ฟใหญ่
    return items
end

--// ============================== สถิติ + แจ้งเตือนแบบรวมชุด ==============================
-- ดูดหลายชิ้นพร้อมกัน จึงรวมแจ้งเตือนเป็นก้อนเดียว ไม่ให้จอเต็มไปด้วย Notification

local Pending = { Count = 0, BestRarity = nil, LastName = nil }

local function markCollected(item, name, rarity)
    if Counted[item] then return end
    Counted[item] = true
    Stats.Count = Stats.Count + 1
    setParagraph(StatusPara, "เก็บสำเร็จแล้ว: " .. tostring(Stats.Count) .. " ชิ้น")

    Pending.Count = Pending.Count + 1
    Pending.LastName = name
    if not Pending.BestRarity
        or (RARITY_RANK[rarity] or 99) < (RARITY_RANK[Pending.BestRarity] or 99) then
        Pending.BestRarity = rarity
    end
end

task.spawn(function()
    while not Fluent.Unloaded do
        task.wait(CONFIG.NOTIFY_BATCH)
        if Pending.Count > 0 then
            if State.NotifyCollect then
                if Pending.Count == 1 then
                    notify("เก็บสำเร็จ", tostring(Pending.LastName) .. " | " .. tostring(Pending.BestRarity), 2)
                else
                    notify("เก็บสำเร็จ " .. Pending.Count .. " ชิ้น",
                        "แรร์สุดในชุดนี้: " .. tostring(Pending.BestRarity), 2)
                end
            end
            Pending.Count, Pending.BestRarity, Pending.LastName = 0, nil, nil
        end
    end
end)

--// ============================== ระบบ ESP ==============================

local function removeESP(item)
    local data = ESPs[item]
    if not data then return end
    ESPs[item] = nil
    pcall(function()
        if data.Conn then data.Conn:Disconnect() end
        if data.Gui then data.Gui:Destroy() end
    end)
end

local function clearAllESP()
    local list = {}
    for item in pairs(ESPs) do table.insert(list, item) end
    for _, item in ipairs(list) do removeESP(item) end
end

local function createESP(item)
    if ESPs[item] then return end

    local adornee = getAnchorPart(item)
    if not adornee then return end

    local gui = Instance.new("BillboardGui")
    gui.Name = "ItemESP"
    gui.Adornee = adornee
    gui.AlwaysOnTop = true
    gui.Size = UDim2.new(0, 260, 0, 36)
    gui.StudsOffset = Vector3.new(0, 3, 0)
    gui.ResetOnSpawn = false

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.GothamBold
    label.TextSize = State.ESPTextSize
    label.TextStrokeTransparency = 0
    label.TextColor3 = Color3.new(1, 1, 1)
    label.Text = "..."
    label.Parent = gui

    local ok = pcall(function()
        gui.Parent = LocalPlayer:WaitForChild("PlayerGui")
    end)
    if not ok then
        gui:Destroy()
        return
    end

    -- ลบป้ายทันทีเมื่อไอเทมถูกเก็บ/ถูกลบ
    local conn = item.AncestryChanged:Connect(function(_, newParent)
        if not newParent then removeESP(item) end
    end)

    ESPs[item] = { Gui = gui, Label = label, Conn = conn }
end

local function updateESP()
    local root = getRoot()
    local alive = {}

    local espRadius = math.max(State.Range * 5, 150)
    for _, item in ipairs(getAllItems()) do
        if item.Parent and root then
            local pos = getItemPosition(item)
            local dist = pos and (pos - root.Position).Magnitude or math.huge
            if pos and dist <= espRadius then
                alive[item] = true
                createESP(item)

                local data = ESPs[item]
                if data then
                    local rarity = getRarity(item)
                    data.Label.Text = string.format("%s | %s | %.0f studs", getItemName(item), rarity, dist)
                    data.Label.TextSize = State.ESPTextSize
                    if State.ESPColorMode == "Custom" then
                        data.Label.TextColor3 = State.ESPColor
                    else
                        data.Label.TextColor3 = RARITY_COLORS[rarity] or RARITY_COLORS.Common
                    end
                end
            end
        end
    end

    local stale = {}
    for item in pairs(ESPs) do
        if not alive[item] then table.insert(stale, item) end
    end
    for _, item in ipairs(stale) do removeESP(item) end
end

--// ============================== กระตุ้นการเก็บ ==============================

-- แตะโซน (TouchInterest) และ/หรือกด ProximityPrompt เพื่อให้เกมรับของเข้ากระเป๋า
local function triggerPickup(root, zone, prompt)
    if zone and firetouchinterest then
        pcall(function()
            firetouchinterest(root, zone, 0)
            firetouchinterest(root, zone, 1)
        end)
    end
    if prompt and fireproximityprompt then
        pcall(fireproximityprompt, prompt)
    end
end

--// ============================== โหมด Pull: ดูดทุกชิ้นพร้อมกัน ==============================
-- แนวคิด: มี "ลูปเดียว" ทำงานทุกเฟรม (Heartbeat) เลื่อนไอเทมทุกชิ้นใน Targets เข้าหาตัวพร้อมกัน
-- ไม่ใช่แยกดูดทีละชิ้น จึงเร็วและได้ทั้งหมดในเฟรมเดียว

local Targets     = {}    -- [item] = { Movable, Zone, Prompt, Name, Rarity, Parts }
local VacConn     = nil
local LastTrigger = 0

-- เพิ่มไอเทมเข้ารายการดูด
local function addTarget(item)
    if Targets[item] then return end
    local movable = getMovable(item)
    if not movable then return end

    local t = {
        Movable = movable,
        Zone    = getZone(item),
        Prompt  = item:FindFirstChildWhichIsA("ProximityPrompt", true),
        Name    = getItemName(item),
        Rarity  = getRarity(item),
        Parts   = {},
    }

    -- ปิดการชนชั่วคราว กันของหลายชิ้นซ้อนตัวแล้วดันตัวละครกระเด็น (จะคืนค่าเมื่อหยุด)
    if State.PullMove then pcall(function()
        local list = movable:IsA("BasePart") and { movable } or {}
        for _, d in ipairs(movable:GetDescendants()) do
            if d:IsA("BasePart") then table.insert(list, d) end
        end
        for _, part in ipairs(list) do
            t.Parts[part] = part.CanCollide
            part.CanCollide = false
        end
    end) end

    Targets[item] = t
end

local function restoreTarget(t)
    for part, canCollide in pairs(t.Parts) do
        pcall(function() part.CanCollide = canCollide end)
    end
end

-- ทำงานทุกเฟรม: ดูดทุกชิ้นพร้อมกัน
local function vacuumStep()
    local root = getRoot()
    if not root then return end

    local rootCF = root.CFrame
    local speed = math.clamp(State.PullSpeed, 0.1, 1)

    local now = os.clock()
    local doTrigger = (now - LastTrigger) >= CONFIG.PULL_TRIGGER
    if doTrigger then LastTrigger = now end

    for item, t in pairs(Targets) do
        if not item:IsDescendantOf(Workspace) then
            -- หายจากพื้น = เข้ากระเป๋าแล้ว
            Targets[item] = nil
            markCollected(item, t.Name, t.Rarity)
        else
            pcall(function()
                if not State.PullMove then
                    return -- ไม่ย้ายของ: แค่สั่งเก็บ (ส่วนกระตุ้นการเก็บอยู่ด้านล่าง)
                elseif speed >= 1 then
                    t.Movable:PivotTo(rootCF) -- ดูดทันที
                else
                    local cur = t.Movable:GetPivot()
                    if (cur.Position - rootCF.Position).Magnitude <= CONFIG.PULL_CLOSE then
                        t.Movable:PivotTo(rootCF)
                    else
                        t.Movable:PivotTo(cur:Lerp(rootCF, speed))
                    end
                end
            end)
            if doTrigger then
                triggerPickup(root, t.Zone, t.Prompt)
            end
        end
    end
end

local WarnedTouch = false
local function startVacuum()
    if not firetouchinterest and not WarnedTouch then
        WarnedTouch = true
        notify("ใช้ไม่ได้", "executor ไม่รองรับ firetouchinterest โหมด Pull จะเก็บของไม่ได้ ให้ใช้โหมด Walk", 8)
    end
    if not VacConn then
        VacConn = RunService.Heartbeat:Connect(function()
            local now = os.clock()
            if now - (Stats.VacLast or 0) < CONFIG.PULL_TRIGGER then return end
            Stats.VacLast = now
            local ok, err = pcall(vacuumStep)
            if not ok then warn("[ItemSystem] vacuum error:", err) end
        end)
    end
end

local function stopVacuum()
    if VacConn then
        VacConn:Disconnect()
        VacConn = nil
    end
    for item, t in pairs(Targets) do
        restoreTarget(t)
        Targets[item] = nil
    end
end

--// ============================== โหมด Walk ==============================

local function moveAndWait(humanoid, position, timeout)
    local done = false
    local conn = humanoid.MoveToFinished:Connect(function() done = true end)
    humanoid:MoveTo(position)

    local t0 = os.clock()
    while not done and (os.clock() - t0) < timeout do
        task.wait()
    end
    conn:Disconnect()
    return done
end

local function walkToPosition(target, isValid)
    local reached = false
    local ok, err = pcall(function()
        local humanoid, root = getHumanoid(), getRoot()
        if not humanoid or not root then return end

        local path = PathfindingService:CreatePath({
            AgentRadius = 2,
            AgentHeight = 5,
            AgentCanJump = true,
        })
        path:ComputeAsync(root.Position, target)

        if path.Status == Enum.PathStatus.Success then
            reached = true
            for _, wp in ipairs(path:GetWaypoints()) do
                if not isValid() then
                    reached = false
                    return
                end
                if wp.Action == Enum.PathWaypointAction.Jump then
                    humanoid.Jump = true
                end
                if not moveAndWait(humanoid, wp.Position, CONFIG.WALK_TIMEOUT) then
                    reached = false
                    break
                end
            end
        else
            reached = moveAndWait(humanoid, target, CONFIG.WALK_TIMEOUT * 2)
        end
    end)

    if not ok then warn("[ItemSystem] Walk error:", err) end
    return reached
end

local function walkToItem(item)
    if Walking then return end
    Walking = true

    local target = getItemPosition(item)
    if target then
        local name, rarity = getItemName(item), getRarity(item)
        local function valid()
            return State.CollectEnabled and State.Mode == "Walk" and item.Parent ~= nil
        end

        walkToPosition(target, valid)

        local root = getRoot()
        if root and item.Parent then
            triggerPickup(root, getZone(item), item:FindFirstChildWhichIsA("ProximityPrompt", true))
            task.wait(0.15)
        end

        if not item:IsDescendantOf(Workspace) then
            markCollected(item, name, rarity)
        elseif item.Parent then
            Skipped[item] = os.clock() + CONFIG.SKIP_TIME -- ยังไม่ถูกเก็บ พักไว้ก่อน
        end
    end

    Walking = false
end

local function walkHome()
    if Walking or not State.Home then return end
    Walking = true

    local home = State.Home
    walkToPosition(home, function()
        return State.CollectEnabled and State.Mode == "Walk"
    end)

    State.Home = nil -- ไม่ว่าถึงหรือไม่ ล้างจุดกลับบ้าน กันวนไม่รู้จบ
    Walking = false
end

--// ============================== ลูปสแกน Auto-Collect ==============================

local function scanOnce()
    local root = getRoot()
    if not root then return end

    -- ---------- โหมด Pull: หาทุกชิ้นในระยะ แล้วโยนเข้ารายการดูดพร้อมกัน ----------
    if State.Mode == "Pull" then
        for _, item in ipairs(getAllItems()) do
            if item.Parent and not Targets[item] and not item:GetAttribute("Locked") then
                local pos = getItemPosition(item)
                if pos and (pos - root.Position).Magnitude <= State.Range then
                    addTarget(item)
                end
            end
        end
        startVacuum()
        return
    end

    -- ---------- โหมด Walk ----------
    -- ขั้นที่ 1: ตรวจจับก่อนว่าไอเทมใกล้ที่สุดในระยะคือชิ้นไหน
    -- (ระหว่างเดินเก็บ วัดจาก "จุดเริ่มต้น" เพื่อไม่ให้หลุดไปไกลเรื่อย ๆ)
    local origin = State.Home or root.Position
    local nearest, nearestDist = nil, math.huge
    local now = os.clock()

    for _, item in ipairs(getAllItems()) do
        if item.Parent then
            local skipUntil = Skipped[item]
            if not (skipUntil and now < skipUntil) then
                local pos = getItemPosition(item)
                if pos then
                    local dist = (pos - origin).Magnitude
                    if dist <= State.Range and dist < nearestDist then
                        nearest, nearestDist = item, dist
                    end
                end
            end
        end
    end

    -- ขั้นที่ 2: เจอของ -> เดินไปเก็บ / ไม่เหลือแล้ว -> เดินกลับที่เดิม
    if not Walking then
        if nearest then
            if not State.Home then State.Home = root.Position end
            task.spawn(walkToItem, nearest)
        elseif State.Home then
            task.spawn(walkHome)
        end
    end
end

--// ============================== โหมดลดกราฟิก (เพิ่ม FPS) ==============================
-- Off = ปกติ | Low = ปิดเงา/เอฟเฟกต์/พาร์ติเคิล | Ultra Low = Low + ซ่อนเท็กซ์เจอร์ (ภาพกาก)
-- เก็บค่าเดิมไว้ก่อนแก้เสมอ จึง "คืนค่าเดิมได้" เมื่อเลือก Off

local Perf = { Level = 0, Conn = nil }
local OrigPart  = setmetatable({}, { __mode = "k" })
local OrigFX    = setmetatable({}, { __mode = "k" })
local OrigDecal = setmetatable({}, { __mode = "k" })
local OrigEnv   = nil

local FX_CLASSES = {
    ParticleEmitter = true, Trail = true, Beam = true,
    Smoke = true, Fire = true, Sparkles = true,
}

local function applyPerf(inst, level)
    pcall(function()
        if inst:IsA("BasePart") then
            if inst:IsA("Terrain") then return end
            if not OrigPart[inst] then
                OrigPart[inst] = { inst.Material, inst.Reflectance, inst.CastShadow }
            end
            inst.Material = Enum.Material.Plastic
            inst.Reflectance = 0
            inst.CastShadow = false
        elseif FX_CLASSES[inst.ClassName] or inst:IsA("PostEffect") then
            if OrigFX[inst] == nil then OrigFX[inst] = inst.Enabled end
            inst.Enabled = false
        elseif level >= 2 and (inst:IsA("Decal") or inst:IsA("Texture")) then
            if OrigDecal[inst] == nil then OrigDecal[inst] = inst.Transparency end
            inst.Transparency = 1
        end
    end)
end

local function restorePerf()
    for part, o in pairs(OrigPart) do
        pcall(function()
            part.Material, part.Reflectance, part.CastShadow = o[1], o[2], o[3]
        end)
    end
    for fx, enabled in pairs(OrigFX) do
        pcall(function() fx.Enabled = enabled end)
    end
    for decal, tr in pairs(OrigDecal) do
        pcall(function() decal.Transparency = tr end)
    end
    table.clear(OrigPart); table.clear(OrigFX); table.clear(OrigDecal)

    if OrigEnv then
        pcall(function() Lighting.GlobalShadows = OrigEnv.GlobalShadows end)
        pcall(function() settings().Rendering.QualityLevel = OrigEnv.Quality end)
        pcall(function()
            local terrain = Workspace:FindFirstChildOfClass("Terrain")
            if terrain then
                terrain.WaterWaveSize = OrigEnv.WaveSize
                terrain.WaterReflectance = OrigEnv.WaterRef
            end
        end)
        OrigEnv = nil
    end
end

local function setPerfLevel(level)
    if Perf.Conn then Perf.Conn:Disconnect(); Perf.Conn = nil end
    Perf.Level = 0
    restorePerf()
    if level <= 0 then return end

    Perf.Level = level

    local terrain = Workspace:FindFirstChildOfClass("Terrain")
    OrigEnv = {
        GlobalShadows = Lighting.GlobalShadows,
        Quality = settings().Rendering.QualityLevel,
        WaveSize = terrain and terrain.WaterWaveSize or 0,
        WaterRef = terrain and terrain.WaterReflectance or 0,
    }
    pcall(function() Lighting.GlobalShadows = false end)
    pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
    pcall(function()
        if terrain then
            terrain.WaterWaveSize = 0
            terrain.WaterReflectance = 0
        end
    end)

    -- ของที่เกิดใหม่ทีหลังก็ปรับให้ด้วย
    Perf.Conn = Workspace.DescendantAdded:Connect(function(inst)
        if Perf.Level > 0 then applyPerf(inst, Perf.Level) end
    end)

    -- ปรับของที่มีอยู่แล้วเป็นชุด ๆ (พักทุก 400 ชิ้น กันกระตุกตอนกดเปิด)
    task.spawn(function()
        local ok, err = pcall(function()
            local myLevel = level
            for _, group in ipairs({ Workspace, Lighting }) do
                local all = group:GetDescendants()
                for i, inst in ipairs(all) do
                    if Perf.Level ~= myLevel then return end
                    applyPerf(inst, myLevel)
                    if i % 400 == 0 then task.wait() end
                end
            end
        end)
        if not ok then warn("[ItemSystem] Perf error:", err) end
    end)
end

--// ============================== ปรับขนาด GUI ==============================
-- Fluent ไม่มีสไลเดอร์ปรับขนาดในตัว จึงเพิ่ม UIScale ให้หน้าต่างหลัก

local function findFluentRoot()
    local ok, root = pcall(function() return Window.Root end)
    if ok and typeof(root) == "Instance" then return root end

    -- ไม่เจอ Window.Root -> หา Frame ที่ใหญ่ที่สุดใน ScreenGui ของ Fluent (คือหน้าต่างหลัก)
    local gui = Fluent.GUI
    if typeof(gui) == "Instance" then
        local best, area = nil, 0
        for _, c in ipairs(gui:GetChildren()) do
            if c:IsA("Frame") then
                local a = c.AbsoluteSize.X * c.AbsoluteSize.Y
                if a > area then best, area = c, a end
            end
        end
        return best
    end
    return nil
end

local function setGuiScale(percent)
    local root = findFluentRoot()
    if not root then
        notify("ปรับขนาดไม่ได้", "หาหน้าต่าง Fluent ไม่เจอ", 3)
        return
    end
    local ui = root:FindFirstChildOfClass("UIScale")
    if not ui then
        ui = Instance.new("UIScale")
        ui.Parent = root
    end
    ui.Scale = percent / 100
end

--// ============================== Player ESP: ดูไอเทมในตัวผู้เล่นคนอื่น ==============================
-- แสดงชื่อไอเทมใต้/เหนือตัวผู้เล่นแต่ละคน สีตามความแรร์ (ทอง=Legendary, ม่วง=Epic ...)
-- อ่านจาก Backpack + ของที่ถืออยู่ ถ้าเกมไม่ส่ง Backpack ของคนอื่นมาให้เครื่องเรา จะเห็นเฉพาะของที่ถือ

local ITEM_FOLDERS = { "gun", "melee", "throwable", "consumable", "farming", "misc", "rod", "fish" }
local WeaponRegistry = {}   -- key ของไอเทม -> { Name, Rarity } โหลดจาก ReplicatedStorage.Items
local PlayerESPs = {}       -- [player] = { Gui, Label, Text }
local RarityCanon = {}
for _, r in ipairs(RARITY_LIST) do RarityCanon[string.lower(r)] = r end

-- สร้าง key ระบุไอเทม (ใช้ Mesh/Texture ก่อน เพราะชื่อของใน Backpack อาจไม่ตรงกับในโฟลเดอร์ Items)
local function toolKey(tool)
    local handle = tool:FindFirstChild("Handle")
    local displayName = tostring(tool:GetAttribute("DisplayName") or tool.Name)
    local itemId = tostring(tool:GetAttribute("ItemId") or tool:GetAttribute("Id") or tool.Name)
    local rarity = tostring(tool:GetAttribute("RarityName") or "Common")

    if handle then
        local mesh = handle:FindFirstChildOfClass("SpecialMesh")
        if mesh and mesh.MeshId ~= "" then
            return mesh.MeshId .. (mesh.TextureId or "") .. "_RARITY_" .. rarity
        end
        if handle:IsA("MeshPart") and handle.MeshId ~= "" then
            return handle.MeshId .. (handle.TextureID or "") .. "_RARITY_" .. rarity
        end
    end
    if itemId ~= "" and itemId ~= tool.Name then
        return "ITEMID_" .. itemId .. "_RARITY_" .. rarity
    end
    return "NAME_" .. displayName .. "_" .. tool.Name .. "_RARITY_" .. rarity
end

-- โหลดข้อมูลไอเทมทั้งหมดจาก ReplicatedStorage.Items (ทำครั้งเดียวตอนเริ่ม)
task.spawn(function()
    pcall(function()
        local items = ReplicatedStorage:WaitForChild("Items", 15)
        if not items then return end
        for _, folderName in ipairs(ITEM_FOLDERS) do
            local folder = items:FindFirstChild(folderName)
            if folder then
                for _, tool in ipairs(folder:GetChildren()) do
                    if tool:IsA("Tool") then
                        WeaponRegistry[toolKey(tool)] = {
                            Name   = tostring(tool:GetAttribute("DisplayName") or tool.Name),
                            Rarity = tostring(tool:GetAttribute("RarityName") or "Common"),
                        }
                    end
                end
            end
        end
    end)
end)

-- ชื่อ + ความแรร์ของ Tool (ไม่เจอในทะเบียน ใช้ค่าจากตัว Tool เอง)
local function getToolInfo(tool)
    local reg = WeaponRegistry[toolKey(tool)]
    if reg then return reg.Name, reg.Rarity end
    return tostring(tool:GetAttribute("DisplayName") or tool.Name),
        tostring(tool:GetAttribute("RarityName") or "Common")
end

-- รวบรวมไอเทมของผู้เล่น (Backpack + ที่ถืออยู่) รวมชิ้นซ้ำเป็น xN
local function getPlayerItems(player)
    local cache = Stats.PlayerItemCache[player]
    local now = os.clock()
    if cache and now < (cache.Next or 0) then
        return cache.Items or {}
    end

    local grouped, order, seen = {}, {}, {}

    local function addTool(tool, held)
        if tool:IsA("Tool") and tool.Name ~= "Fists" and not seen[tool] then
            seen[tool] = true
            local name, rarity = getToolInfo(tool)
            local canon = RarityCanon[string.lower(rarity)]
            local key = name .. "|" .. rarity
            local g = grouped[key]
            if not g then
                g = { Name = name, Rarity = canon or rarity, Canon = canon, Count = 0, Held = false }
                grouped[key] = g
                table.insert(order, g)
            end
            g.Count = g.Count + 1
            if held then g.Held = true end
        end
    end

    local bp = player:FindFirstChild("Backpack")
    if bp then
        for _, t in ipairs(bp:GetChildren()) do addTool(t, false) end
    end
    local char = player.Character
    if char then
        for _, t in ipairs(char:GetChildren()) do addTool(t, true) end
    end

    table.sort(order, function(a, b)
        local ra, rb = RARITY_RANK[a.Rarity] or 99, RARITY_RANK[b.Rarity] or 99
        if ra ~= rb then return ra < rb end
        return a.Name < b.Name
    end)

    Stats.PlayerItemCache[player] = { Items = order, Next = now + 0.9 }
    return order
end

local function colorHex(c)
    return string.format("#%02X%02X%02X",
        math.floor(c.R * 255 + 0.5), math.floor(c.G * 255 + 0.5), math.floor(c.B * 255 + 0.5))
end

local function escapeRich(text)
    return (tostring(text):gsub("&", "&amp;"):gsub("<", "&lt;"):gsub(">", "&gt;"))
end

local function removePlayerESP(player)
    local data = PlayerESPs[player]
    if not data then return end
    PlayerESPs[player] = nil
    pcall(function() data.Gui:Destroy() end)
end

local function clearAllPlayerESP()
    local list = {}
    for player in pairs(PlayerESPs) do table.insert(list, player) end
    for _, player in ipairs(list) do removePlayerESP(player) end
end

local function createPlayerESP(player, root)
    local gui = Instance.new("BillboardGui")
    gui.Name = "PlayerItemESP"
    gui.Adornee = root
    gui.AlwaysOnTop = true
    gui.Size = UDim2.new(0, 300, 0, 400)
    gui.StudsOffset = Vector3.new(0, 3, 0)
    gui.ResetOnSpawn = false

    -- ตัวหนังสือชิดขอบล่างของกล่อง เพิ่มบรรทัดแล้วขยายขึ้นด้านบน ไม่ทับตัวผู้เล่น
    local label = Instance.new("TextLabel")
    label.AnchorPoint = Vector2.new(0.5, 1)
    label.Position = UDim2.new(0.5, 0, 0.5, 0)
    label.Size = UDim2.new(1, 0, 0, 0)
    label.AutomaticSize = Enum.AutomaticSize.Y
    label.BackgroundTransparency = 1
    label.RichText = true
    label.Font = Enum.Font.GothamBold
    label.TextSize = State.PETextSize
    label.TextStrokeTransparency = 0.3
    label.TextYAlignment = Enum.TextYAlignment.Bottom
    label.TextColor3 = Color3.new(1, 1, 1)
    label.Text = ""
    label.Parent = gui

    local ok = pcall(function() gui.Parent = LocalPlayer:WaitForChild("PlayerGui") end)
    if not ok then
        gui:Destroy()
        return nil
    end

    local data = { Gui = gui, Label = label, Text = "" }
    PlayerESPs[player] = data
    return data
end

local function updatePlayerESP()
    local myRoot = getRoot()
    local alive = {}

    if myRoot then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local char = player.Character
                local root = char and char:FindFirstChild("HumanoidRootPart")
                local humanoid = char and char:FindFirstChildOfClass("Humanoid")
                if root and humanoid and humanoid.Health > 0 then
                    local dist = (root.Position - myRoot.Position).Magnitude
                    if dist <= State.PERange then
                        -- สร้างข้อความ: หัวข้อ (ชื่อ+ระยะ) + รายการไอเทมที่ผ่านตัวกรอง
                        local lines, shown, hidden = {}, 0, 0
                        local best = nil

                        for _, g in ipairs(getPlayerItems(player)) do
                            -- ความแรร์ที่ไม่รู้จักจะแสดงเสมอ ที่รู้จักจะแสดงตามตัวกรอง
                            local pass = (g.Canon == nil) or State.PERarityFilter[string.lower(g.Canon)]
                            if pass then
                                if shown < State.PEMaxLines then
                                    shown = shown + 1
                                    local color = RARITY_COLORS[g.Rarity] or Color3.new(1, 1, 1)
                                    local text = "[" .. escapeRich(g.Name) .. "]"
                                    if g.Count > 1 then text = text .. " x" .. g.Count end
                                    if State.PEShowHeld and g.Held then text = "▶ " .. text end
                                    table.insert(lines, '<font color="' .. colorHex(color) .. '">' .. text .. "</font>")
                                else
                                    hidden = hidden + 1
                                end
                                if not best or (RARITY_RANK[g.Rarity] or 99) < (RARITY_RANK[best] or 99) then
                                    best = g.Rarity
                                end
                            end
                        end
                        if hidden > 0 then table.insert(lines, "... +" .. hidden) end

                        if State.PEShowName then
                            local headColor = Color3.new(1, 1, 1)
                            if State.PEHeaderByRarity and best then
                                headColor = RARITY_COLORS[best] or headColor
                            end
                            table.insert(lines, 1, string.format('<font color="%s">%s  [%d studs]</font>',
                                colorHex(headColor), escapeRich(player.DisplayName), dist))
                        end

                        if #lines > 0 then
                            alive[player] = true
                            local data = PlayerESPs[player]
                            if data and data.Gui.Adornee ~= root then
                                removePlayerESP(player) -- ตัวละครเกิดใหม่ สร้างป้ายใหม่
                                data = nil
                            end
                            data = data or createPlayerESP(player, root)
                            if data then
                                local text = table.concat(lines, "\n")
                                if data.Text ~= text then
                                    data.Text = text
                                    data.Label.Text = T(text)
                                end
                                data.Label.TextSize = State.PETextSize
                                data.Gui.MaxDistance = State.PERange
                            end
                        end
                    end
                end
            end
        end
    end

    local stale = {}
    for player in pairs(PlayerESPs) do
        if not alive[player] then table.insert(stale, player) end
    end
    for _, player in ipairs(stale) do removePlayerESP(player) end
end

local ItemESPPlayerRemovingConnection = Players.PlayerRemoving:Connect(function(player)
    Stats.PlayerItemCache[player] = nil
    removePlayerESP(player)
end)

-- Player lifecycle: เข้า / ตาย / เกิดใหม่ ให้ Player ESP ตามเองโดยไม่ต้องปิด-เปิด
local PlayerLifecycle = {}
local DeathMarkers = {}

local function removeDeathMarker(player)
    local part = DeathMarkers[player]
    DeathMarkers[player] = nil
    if part then pcall(function() part:Destroy() end) end
end

local function createDeathMarker(player, position)
    if not position then return end
    removeDeathMarker(player)
    local part = Instance.new("Part")
    part.Name = "PlayerESP_DeathMarker"
    part.Size = Vector3.new(0.2, 0.2, 0.2)
    part.Transparency = 1
    part.Anchored = true
    part.CanCollide = false
    part.CanQuery = false
    part.CanTouch = false
    part.CFrame = CFrame.new(position)
    part.Parent = Workspace

    local gui = Instance.new("BillboardGui")
    gui.Name = "DeadESP"
    gui.Adornee = part
    gui.AlwaysOnTop = true
    gui.Size = UDim2.fromOffset(260, 40)
    gui.StudsOffset = Vector3.new(0, 2.5, 0)
    gui.Parent = part

    local label = Instance.new("TextLabel")
    label.Size = UDim2.fromScale(1, 1)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.GothamBold
    label.TextSize = 14
    label.TextStrokeTransparency = 0
    label.TextColor3 = Color3.fromRGB(255, 80, 80)
    label.Text = "☠ " .. tostring(player.DisplayName) .. " [DEAD]"
    label.Parent = gui

    DeathMarkers[player] = part
    task.delay(10, function()
        if DeathMarkers[player] == part then removeDeathMarker(player) end
    end)
end

local function bindPlayerLifecycle(player)
    -- [FIXED] รีเซ็ต handlers เก่าทั้งหมดก่อนผูก lifecycle ใหม่
    if PlayerLifecycle[player] then
        for _, c in pairs(PlayerLifecycle[player]) do
            if typeof(c) == "RBXScriptConnection" then pcall(function() c:Disconnect() end) end
        end
    end
    PlayerLifecycle[player] = { DiedConnection = nil }

    local function bindCharacter(char)
        if not char then return end
        Stats.PlayerItemCache[player] = nil
        removePlayerESP(player)

        -- [FIXED] มี Died handler ได้เพียงหนึ่งตัวต่อผู้เล่น ไม่สะสมทุกครั้งที่เกิดใหม่
        local previous = PlayerLifecycle[player] and PlayerLifecycle[player].DiedConnection
        if previous then pcall(function() previous:Disconnect() end) end
        if PlayerLifecycle[player] then PlayerLifecycle[player].DiedConnection = nil end

        local humanoid = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 5)
        if not humanoid or player.Character ~= char then return end
        PlayerLifecycle[player].DiedConnection = humanoid.Died:Connect(function()
            local root = char:FindFirstChild("HumanoidRootPart")
            if root then createDeathMarker(player, root.Position) end
            removePlayerESP(player)
            Stats.PlayerItemCache[player] = nil
        end)
    end

    if player.Character then bindCharacter(player.Character) end
    table.insert(PlayerLifecycle[player], player.CharacterAdded:Connect(function(char)
        removeDeathMarker(player)
        removePlayerESP(player)
        Stats.PlayerItemCache[player] = nil
        task.wait(0.15)
        if player.Character == char then bindCharacter(char) end
    end))
    table.insert(PlayerLifecycle[player], player.CharacterRemoving:Connect(function(char)
        -- [FIXED] ลบ label ทันทีที่ร่างเดิมหาย ไม่ต้องรอรอบอัปเดต ESP
        local data = PlayerESPs[player]
        if data and data.Gui and data.Gui.Adornee and data.Gui.Adornee:IsDescendantOf(char) then
            removePlayerESP(player)
        end
        Stats.PlayerItemCache[player] = nil
    end))
end

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then bindPlayerLifecycle(player) end
end
local PlayerLifecycleAddedConnection = Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then bindPlayerLifecycle(player) end
end)
local PlayerLifecycleRemovingConnection = Players.PlayerRemoving:Connect(function(player)
    removeDeathMarker(player)
    local conns = PlayerLifecycle[player]
    PlayerLifecycle[player] = nil
    if conns then for _, c in pairs(conns) do pcall(function() c:Disconnect() end) end end
end)

-- ตัวเรา: แจ้งเตือน Discord ตอนตาย และผูก Humanoid ใหม่เมื่อเกิดใหม่
local LocalLifecycle = nil          -- ตัวเก็บ connection ของตัวละครปัจจุบัน (มีเมธอด Disconnect)
local LocalCharacterRemoving = nil
local LastDeathNoticeAt = -100
local BoundLocalCharacter = nil

local function discordDeathNotice(char, hum, source)
    -- ส่งเฉพาะเมื่อเปิดสวิตช์ และมี URL
    if not State.DeathWebhook then return end
    if tostring(CONFIG.DISCORD_WEBHOOK_URL or ""):gsub("%s+", "") == "" then return end

    local now = os.clock()
    if now - LastDeathNoticeAt < 3 then return end   -- กันส่งซ้ำจากหลายสัญญาณ
    LastDeathNoticeAt = now

    -- เก็บข้อมูลทันที เพราะหลังตายของ/ตัวละครจะหายไป
    local function clip(text, n)
        text = tostring(text)
        if #text > n then return text:sub(1, n - 3) .. "..." end
        return text
    end

    local root = char and char:FindFirstChild("HumanoidRootPart")
    local pos = root and root.Position

    local reason = "Humanoid.Died"
    if hum then
        local attrReason = hum:GetAttribute("DeathReason")
        if attrReason ~= nil and tostring(attrReason) ~= "" then
            reason = tostring(attrReason)
        elseif hum.Health <= 0 then
            reason = "Health <= 0"
        end
    end
    if source then reason = reason .. " | " .. tostring(source) end

    local items = {}
    pcall(function()
        local seen = {}
        local function add(tool)
            if tool:IsA("Tool") and tool.Name ~= "Fists" and not seen[tool] then
                seen[tool] = true
                local name = tostring(tool:GetAttribute("DisplayName") or tool.Name)
                local rarity = tool:GetAttribute("RarityName")
                table.insert(items, rarity and (name .. " [" .. tostring(rarity) .. "]") or name)
            end
        end
        local bp = LocalPlayer:FindFirstChild("Backpack")
        if bp then for _, t in ipairs(bp:GetChildren()) do add(t) end end
        if char then for _, t in ipairs(char:GetChildren()) do add(t) end end
    end)

    local nearby = {}
    if pos then
        pcall(function()
            for _, pl in ipairs(Players:GetPlayers()) do
                if pl ~= LocalPlayer then
                    local r = pl.Character and pl.Character:FindFirstChild("HumanoidRootPart")
                    if r then
                        local d = (r.Position - pos).Magnitude
                        if d <= 150 then table.insert(nearby, { pl.DisplayName, d }) end
                    end
                end
            end
        end)
        table.sort(nearby, function(x, y) return x[2] < y[2] end)
    end
    local nearbyLines = {}
    for i, n in ipairs(nearby) do
        if i > 8 then break end
        table.insert(nearbyLines, string.format("%s (%d studs)", n[1], n[2]))
    end

    local fields = {
        { name = "Player", value = tostring(LocalPlayer.DisplayName) .. " (" .. tostring(LocalPlayer.Name) .. ")", inline = false },
        { name = "Reason", value = clip(reason, 200), inline = true },
        { name = "PlaceId", value = tostring(game.PlaceId), inline = true },
        { name = "ผู้เล่นใกล้ตัว (≤150 studs)", value = clip(#nearbyLines > 0 and table.concat(nearbyLines, "\n") or "ไม่มี", 1000), inline = false },
        { name = "ของในตัวตอนตาย (" .. #items .. ")", value = clip(#items > 0 and table.concat(items, "\n") or "ไม่มี", 1000), inline = false },
        { name = "JobId", value = tostring(game.JobId ~= "" and game.JobId or "Private/Studio"), inline = false },
    }
    if pos then
        table.insert(fields, 3, { name = "Position", value = string.format("%.1f, %.1f, %.1f", pos.X, pos.Y, pos.Z), inline = false })
    end

    task.spawn(function()
        local ok, err = sendDiscordWebhook("☠ ผู้เล่นตาย", "ตรวจพบว่าตัวละครของคุณตาย/ถูกรีเซ็ต", fields)
        if not ok then
            warn("[ItemSystem][Discord] ส่งแจ้งเตือนตายไม่สำเร็จ:", err)
            notify("Discord", "ส่ง Death Webhook ไม่สำเร็จ: " .. tostring(err), 7)
        end
    end)
end

local function bindLocalCharacter(char)
    if not char or BoundLocalCharacter == char then return end
    BoundLocalCharacter = char

    if LocalLifecycle then pcall(function() LocalLifecycle:Disconnect() end) end
    if LocalCharacterRemoving then pcall(function() LocalCharacterRemoving:Disconnect() end) end
    LocalCharacterRemoving = nil

    local conns = {}
    LocalLifecycle = {
        Disconnect = function()
            for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
            table.clear(conns)
        end,
    }

    local hum = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 15)
    if hum then
        -- ฟังหลายสัญญาณ: เกมนี้อาจไม่ยิง Died ทุกครั้ง (ใช้ IsDead / Health แทน)
        table.insert(conns, hum.Died:Connect(function() discordDeathNotice(char, hum, "Died") end))
        table.insert(conns, hum.HealthChanged:Connect(function(h)
            if h <= 0 then discordDeathNotice(char, hum, "Health<=0") end
        end))
        table.insert(conns, hum:GetAttributeChangedSignal("IsDead"):Connect(function()
            if hum:GetAttribute("IsDead") == true then discordDeathNotice(char, hum, "IsDead") end
        end))
    end

    -- Reset/respawn บางกรณี executor เห็น CharacterRemoving ก่อน Died จึงใช้เป็น fallback
    LocalCharacterRemoving = LocalPlayer.CharacterRemoving:Connect(function(removingChar)
        if removingChar ~= char then return end
        local dead = false
        if hum then
            dead = hum.Health <= 0 or hum:GetAttribute("IsDead") == true
        end
        if dead then discordDeathNotice(char, hum, "CharacterRemoving") end
    end)
end

if LocalPlayer.Character then task.spawn(bindLocalCharacter, LocalPlayer.Character) end
local LocalCharacterAddedConnection = LocalPlayer.CharacterAdded:Connect(function(char)
    task.spawn(function()
        task.wait(0.1)
        if Fluent.Unloaded then return end
        bindLocalCharacter(char)
    end)
end)
task.spawn(function()
    while not Fluent.Unloaded do task.wait(0.5) end
    if LocalCharacterAddedConnection then
        pcall(function() LocalCharacterAddedConnection:Disconnect() end)
        LocalCharacterAddedConnection = nil
    end
    if LocalCharacterRemoving then
        pcall(function() LocalCharacterRemoving:Disconnect() end)
        LocalCharacterRemoving = nil
    end
end)

--// ============================== Combat: ไม่มีแรงสะท้อน + สเตมิน่าไม่จำกัด ==============================
-- จากโค้ดเกม: แรงสะท้อนมาจากแอตทริบิวต์ "Recoil" บน Tool ปืน (ModuleScript ShoulderCamera อ่านตอนยิง)
--             สเตมิน่าเป็นค่า 0..1 ในโมดูล ReplicatedStorage.Modules.Game.Sprint (sprint_bar)
-- ทั้งสองอย่างแก้ที่ฝั่งเครื่องเราเท่านั้น ไม่ได้ยิงรีโมตอะไรเอง

local GameModules = {}

local function loadGameModule(name)
    if GameModules[name] ~= nil then return GameModules[name] or nil end
    local ok, mod = pcall(function()
        local modules = ReplicatedStorage:FindFirstChild("Modules")
        local game_ = modules and modules:FindFirstChild("Game")
        local inst = game_ and game_:FindFirstChild(name)
        if not inst then error("not found") end
        return require(inst)
    end)
    if ok and type(mod) == "table" then
        GameModules[name] = mod
        return mod
    end
    return nil -- ไม่เก็บค่า false ไว้ เผื่อโมดูลโหลดช้ากว่าสคริปต์ เดี๋ยวลองใหม่รอบหน้า
end

local TrackedGuns = {} -- [tool] = { Orig = ค่า Recoil เดิม, Conn = การดักการเปลี่ยนค่า }

local function hookGun(tool)
    if TrackedGuns[tool] or not tool:IsA("Tool") then return end
    local current = tool:GetAttribute("Recoil")
    if type(current) ~= "number" then return end -- ไม่ใช่ปืนที่มีแรงสะท้อน

    local data = { Orig = current }
    TrackedGuns[tool] = data
    if current ~= 0 then tool:SetAttribute("Recoil", 0) end

    -- ถ้าเกมตั้งค่าใหม่ (เช่น ใส่/ถอดอุปกรณ์เสริม) ให้จำค่านั้นไว้แล้วกดเป็น 0 อีกรอบ
    data.Conn = tool:GetAttributeChangedSignal("Recoil"):Connect(function()
        local v = tool:GetAttribute("Recoil")
        if State.NoRecoil and type(v) == "number" and v ~= 0 then
            data.Orig = v
            tool:SetAttribute("Recoil", 0)
        end
    end)
end

local function scanGuns()
    local bp = LocalPlayer:FindFirstChild("Backpack")
    if bp then
        for _, t in ipairs(bp:GetChildren()) do hookGun(t) end
    end
    local char = LocalPlayer.Character
    if char then
        for _, t in ipairs(char:GetChildren()) do hookGun(t) end
    end
end

-- [FIXED] ผูก ChildAdded ของ Backpack/Character เพื่อให้ปืนที่ได้ใหม่หรือหลังเกิดใหม่ถูกตั้ง Recoil=0 ทันที
local GunContainerConnections = setmetatable({}, { __mode = "k" })
local CombatLifecycleConnections = {}
local function watchGunContainer(container)
    if not container or GunContainerConnections[container] then return end
    GunContainerConnections[container] = container.ChildAdded:Connect(function(child)
        if State.NoRecoil and child and child:IsA("Tool") then
            task.defer(function()
                if State.NoRecoil and child.Parent then pcall(hookGun, child) end
            end)
        end
    end)
end
local function unwatchGunContainer(container)
    local conn = container and GunContainerConnections[container]
    if conn then
        pcall(function() conn:Disconnect() end)
        GunContainerConnections[container] = nil
    end
end
local function bindGunContainers()
    watchGunContainer(LocalPlayer:FindFirstChild("Backpack"))
    watchGunContainer(LocalPlayer.Character)
    if State.NoRecoil then pcall(scanGuns) end
end

table.insert(CombatLifecycleConnections, LocalPlayer.CharacterAdded:Connect(function(char)
    task.wait(0.15)
    if LocalPlayer.Character == char then
        watchGunContainer(char)
        if State.NoRecoil then pcall(scanGuns) end
    end
end))
table.insert(CombatLifecycleConnections, LocalPlayer.CharacterRemoving:Connect(function(char)
    -- [FIXED] ถอด listener ของร่างเก่าทันทีเพื่อลด connection ค้างหลังตาย/เกิดใหม่
    unwatchGunContainer(char)
end))
table.insert(CombatLifecycleConnections, LocalPlayer.ChildAdded:Connect(function(child)
    if child.Name == "Backpack" then
        watchGunContainer(child)
        if State.NoRecoil then pcall(scanGuns) end
    end
end))
table.insert(CombatLifecycleConnections, LocalPlayer.ChildRemoved:Connect(function(child)
    if child.Name == "Backpack" then unwatchGunContainer(child) end
end))
bindGunContainers()

local function restoreGuns()
    for tool, data in pairs(TrackedGuns) do
        if data.Conn then data.Conn:Disconnect() end
        pcall(function()
            if tool.Parent and data.Orig ~= nil then tool:SetAttribute("Recoil", data.Orig) end
        end)
    end
    TrackedGuns = {}
end

-- ตรวจจับระบบเกมก่อนใช้งาน (เผื่อแมพ/เกมอัปเดตแล้วโครงสร้างเปลี่ยน)
local function detectSystems()
    local gunCount, sample = 0, nil
    local function countIn(container)
        if not container then return end
        for _, t in ipairs(container:GetChildren()) do
            if t:IsA("Tool") and type(t:GetAttribute("Recoil")) == "number" then
                gunCount = gunCount + 1
                sample = sample or (t.Name .. " Recoil=" .. tostring(t:GetAttribute("Recoil")))
            end
        end
    end
    countIn(LocalPlayer:FindFirstChild("Backpack"))
    countIn(LocalPlayer.Character)

    local sprint = loadGameModule("Sprint")
    local sprintOk = sprint ~= nil and type(sprint.sprint_bar) == "table"
    local cam = loadGameModule("ShoulderCamera")
    local camOk = cam ~= nil and cam.recoil ~= nil

    return {
        PlaceId   = game.PlaceId,
        Sprint    = sprintOk,
        Camera    = camOk,
        Guns      = gunCount,
        GunSample = sample,
        MaxStamina = LocalPlayer:GetAttribute("MaxStamina"),
    }
end

local function describeSystems(info)
    return string.format(
        "PlaceId: %s\nระบบสเตมิน่า (Sprint): %s%s\nกล้อง/แรงสะท้อน (ShoulderCamera): %s\nปืนที่พบในตัว (มี Recoil): %d%s",
        tostring(info.PlaceId),
        info.Sprint and "พบ" or "ไม่พบ",
        info.MaxStamina and (" (MaxStamina " .. tostring(info.MaxStamina) .. ")") or "",
        info.Camera and "พบ" or "ไม่พบ",
        info.Guns,
        info.GunSample and (" เช่น " .. info.GunSample) or "")
end

-- ลูปประหยัดทรัพยากร: เติมสเตมิน่า/เคลียร์แรงสะท้อนทุก 0.1 วิ และสแกนปืนใหม่ทุก ~0.25 วิ
task.spawn(function()
    local acc = 0
    while not Fluent.Unloaded do
        local dt = task.wait(0.10)
        acc = acc + dt

        if State.InfStamina then
            local sprint = loadGameModule("Sprint")
            if sprint and sprint.sprint_bar then
                pcall(function()
                    if sprint.sprint_bar.get() < 1 then sprint.sprint_bar.set(1) end
                end)
            end
        end

        if State.NoRecoil then
            local cam = loadGameModule("ShoulderCamera")
            if cam and cam.recoil then
                pcall(function() cam.recoil.r = CFrame.new() end)
            end
            if acc >= 0.25 then pcall(scanGuns) end
        end

        if acc >= 0.25 then acc = 0 end
    end
end)

--// ============================== มุดดิน (Burrow) ==============================
-- คอมกดปุ่มที่ตั้งไว้ (ค่าเริ่มต้น Z) / มือถือกดปุ่ม Snap : ย้ายตัวละครลงใต้พื้นตามระยะที่ตั้ง
-- ใต้ดินมีแผ่นรองโปร่งใสตามตัวไป (เดินใต้ดินได้) และปิดการชนของตัวละคร กดอีกครั้งเพื่อโผล่ขึ้นมา
-- ตำแหน่งนี้ถูกส่งไปที่เซิร์ฟเวอร์จริง จึงมีความเสี่ยงโดนระบบของเกมตรวจจับ
do
    local UserInputService = game:GetService("UserInputService")
    local B = {
        Enabled = false,
        Depth = 10,
        Active = false,
        ShowButton = (Ext.platform == "mobile"),
        KeyName = "Z",
        KeyCode = Enum.KeyCode.Z,
    }
    Ext.Burrow = B

    local platform, noclipConn, diedConn
    local platformY, feetOffset, origY, startChar
    local btnGui, btn

    local function refreshButton()
        if btnGui then btnGui.Enabled = B.Enabled and B.ShowButton end
        if btn then btn.Text = B.Active and "Snap ▲" or "Snap ▼" end
    end

    -- เก็บกวาดสถานะ (ไม่ย้ายตัวละคร)
    local function clearState()
        if noclipConn then
            noclipConn:Disconnect()
            noclipConn = nil
        end
        if diedConn then
            diedConn:Disconnect()
            diedConn = nil
        end
        if platform then
            platform:Destroy()
            platform = nil
        end
        B.Active = false
        startChar = nil
        local root = getRoot()
        if root then root.CanCollide = true end
        refreshButton()
    end

    local function down()
        if B.Active then return end
        local char, root, hum = LocalPlayer.Character, getRoot(), getHumanoid()
        if not (char and root and hum) or hum.Health <= 0 then
            notify("มุดดิน", "ตัวละครยังไม่พร้อม", 3)
            return
        end

        local depth = math.clamp(B.Depth, 2, 150)
        local y = root.Position.Y
        if y - depth < -400 then depth = math.max(2, y + 400) end -- กันตกต่ำกว่าเขตทำลายของแมพ
        origY = y
        feetOffset = hum.HipHeight + root.Size.Y / 2
        local targetY = y - depth
        platformY = targetY - feetOffset - 1

        platform = Instance.new("Part")
        platform.Name = "IMS_BurrowFloor"
        platform.Anchored = true
        platform.CanCollide = true
        platform.Transparency = 1
        platform.Size = Vector3.new(80, 2, 80)
        platform.CFrame = CFrame.new(root.Position.X, platformY, root.Position.Z)
        platform.Parent = Workspace

        local rot = root.CFrame - root.CFrame.Position
        root.CFrame = CFrame.new(root.Position.X, targetY, root.Position.Z) * rot
        root.AssemblyLinearVelocity = Vector3.zero

        B.Active = true
        startChar = char

        noclipConn = RunService.Stepped:Connect(function()
            local c, r = LocalPlayer.Character, getRoot()
            if c ~= startChar or not r then
                clearState()
                return
            end
            for _, part in ipairs(c:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
            if platform then
                platform.CFrame = CFrame.new(r.Position.X, platformY, r.Position.Z)
            end
            -- ตกลงไปลึกกว่าที่ตั้ง -> ดึงกลับมาที่ระดับเดิม
            if r.Position.Y < platformY - 8 then
                r.CFrame = CFrame.new(r.Position.X, platformY + feetOffset + 2, r.Position.Z)
                r.AssemblyLinearVelocity = Vector3.zero
            end
        end)
        diedConn = hum.Died:Connect(clearState)

        notify("มุดดิน", "มุดลงแล้ว ลึก " .. math.floor(depth) .. " studs กดอีกครั้งเพื่อขึ้น", 4)
        refreshButton()
    end

    local function up()
        if not B.Active then return end
        local root, hum = getRoot(), getHumanoid()
        if root then
            local x, z = root.Position.X, root.Position.Z
            local params = RaycastParams.new()
            params.FilterType = Enum.RaycastFilterType.Exclude
            params.FilterDescendantsInstances = { LocalPlayer.Character, platform }

            -- หาพื้นด้านบนที่ตำแหน่ง x,z ปัจจุบัน เพื่อโผล่ขึ้นมาบนพื้นไม่ใช่ในกำแพง
            local hit = Workspace:Raycast(Vector3.new(x, origY + 3, z), Vector3.new(0, -300, 0), params)
            if not hit then
                hit = Workspace:Raycast(Vector3.new(x, origY + 150, z), Vector3.new(0, -400, 0), params)
            end
            local feet = hum and (hum.HipHeight + root.Size.Y / 2) or 3
            local newY = hit and (hit.Position.Y + math.max(feet, 3) + 1) or origY
            root.CFrame = CFrame.new(x, newY, z) * (root.CFrame - root.CFrame.Position)
            root.AssemblyLinearVelocity = Vector3.zero
        end
        clearState()
        notify("มุดดิน", "ขึ้นมาแล้ว", 3)
    end

    function B.toggle()
        if not B.Enabled then return end
        if B.Active then up() else down() end
    end

    function B.setEnabled(value)
        B.Enabled = value and true or false
        if not B.Enabled and B.Active then up() end
        refreshButton()
    end

    function B.setShowButton(value)
        B.ShowButton = value and true or false
        refreshButton()
    end

    function B.setKey(name)
        local ok, kc = pcall(function() return Enum.KeyCode[name] end)
        if ok and kc then
            B.KeyName = name
            B.KeyCode = kc
        end
    end

    function Ext.burrowCleanup()
        if B.Active then pcall(up) end
        pcall(clearState)
        if btnGui then
            btnGui:Destroy()
            btnGui = nil
            btn = nil
        end
    end

    -- ปุ่ม Snap บนหน้าจอ (มือถือ) แตะ = มุด/ขึ้น  ลาก = ย้ายตำแหน่ง
    local okBtn = pcall(function()
        btnGui = Ext.makeGui("IMS_Snap", 90)
        btnGui.Enabled = false

        btn = Instance.new("TextButton")
        btn.Name = "SnapButton"
        btn.Size = UDim2.fromOffset(76, 76)
        btn.Position = UDim2.new(1, -120, 0.55, 0)
        btn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
        btn.BackgroundTransparency = 0.1
        btn.TextColor3 = Color3.new(1, 1, 1)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 16
        btn.Text = "Snap ▼"
        btn.AutoButtonColor = false
        btn.Active = true
        btn.Parent = btnGui

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1, 0)
        corner.Parent = btn
        local stroke = Instance.new("UIStroke")
        stroke.Color = Color3.fromRGB(120, 170, 255)
        stroke.Thickness = 2
        stroke.Parent = btn

        Ext.makeTappable(btn, function()
            if not B.Enabled then
                notify("มุดดิน", "กรุณาเปิดระบบมุดดินก่อน", 3)
                return
            end
            B.toggle()
        end)
    end)
    if not okBtn then warn("[ItemSystem] สร้างปุ่ม Snap ไม่สำเร็จ") end

    -- [FIXED] เก็บ input connection ที่ผูกกับ UserInputService ซึ่งไม่ถูกลบตาม GUI
    B.InputConnection = UserInputService.InputBegan:Connect(function(input, processed)
        if Fluent.Unloaded then return end
        if processed then return end
        if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == B.KeyCode then
            B.toggle()
        end
    end)
    task.spawn(function()
        while not Fluent.Unloaded do task.wait(0.5) end
        if B.InputConnection then
            pcall(function() B.InputConnection:Disconnect() end)
            B.InputConnection = nil
        end
    end)
end

-- [FIXED] ปิด Fish Log / Fishing Runtime ในรุ่นที่เน้น Combat, ESP และ Aimbot
-- Stub นี้ไว้รองรับ Config เก่าที่อาจอ่าน Ext.Fish.WebhookRarities โดยไม่ให้เกิด nil reference
Ext.Fish = { WebhookRarities = nil, Enabled = false }

--// ============================== Debug ==============================

local function runDebug()
    local root = getRoot()
    local folder = findDropFolder()
    local items = getAllItems()

    local lines = {}
    table.insert(lines, "โฟลเดอร์: " .. (folder and folder:GetFullName() or "ไม่พบ (ใช้ค้นหาสำรอง)"))
    table.insert(lines, "จำนวนไอเทมที่เห็น: " .. tostring(#items))

    local nearestText, best = "-", math.huge
    for _, item in ipairs(items) do
        local pos = getItemPosition(item)
        local dist = (pos and root) and (pos - root.Position).Magnitude or -1
        local info = string.format("[%s] %s | rarity=%s | zone=%s | dist=%.1f",
            item.ClassName, item.Name, getRarity(item), tostring(getZone(item) ~= nil), dist)
        print("[ItemSystem][Debug]", info)
        if dist >= 0 and dist < best then
            best, nearestText = dist, info
        end
    end
    table.insert(lines, "ใกล้สุด: " .. nearestText)
    table.insert(lines, "ดูรายละเอียดเต็มใน Console (F9)")

    notify("Debug", table.concat(lines, "\n"), 8)
end

--// ============================== สร้าง UI (Fluent) ==============================
local winW, winH, tabW = 580, 460, 160
if Ext.platform == "mobile" then
    local cam = Workspace.CurrentCamera
    local vp = cam and cam.ViewportSize or Vector2.new(800, 400)
    winW = math.clamp(math.floor(vp.X * 0.82), 320, 520)
    winH = math.clamp(math.floor(vp.Y * 0.8), 220, 400)
    tabW = 110
end

Window = Fluent:CreateWindow({
    Title = "Item Management System v18",
    SubTitle = "General + Stamina Fix + Player Exclude",
    TabWidth = tabW,
    Size = UDim2.fromOffset(winW, winH),
    Acrylic = false, -- ปิดเบลอ (เบลออาจถูกตรวจจับได้และกินเครื่อง)
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.RightControl,
})

local Tabs = {
    Main      = Window:AddTab({ Title = "Main", Icon = "" }),
    PlayerESP = Window:AddTab({ Title = "Player ESP", Icon = "" }),
    Combat    = Window:AddTab({ Title = "Combat", Icon = "" }),
    Aimbot    = Window:AddTab({ Title = "Aimbot", Icon = "" }),
    Movement  = Window:AddTab({ Title = "Movement", Icon = "" }),
    General   = Window:AddTab({ Title = "General", Icon = "" }),
    Settings  = Window:AddTab({ Title = "Settings", Icon = "settings" }),
}

local Options = Fluent.Options

do
-- [v18] Crate skip: ใช้ API ของเกมเฉพาะเมื่อบัญชีมีสิทธิ์ที่เกมกำหนด
-- ใน Crate.skip_spin() ของเกมมีการตรวจ Data.can_skip_crate_animation และเปิดหน้าซื้อถ้าไม่มีสิทธิ์
-- จึงไม่เรียก API อัตโนมัติเมื่อไม่มีสิทธิ์ และไม่ข้ามระบบซื้อของของเกม
local CrateSkipConfig = { Enabled = false, LastSpinning = false, LastCall = 0, Warned = false }
local CrateModule = nil
local CrateSkipData = nil
local CrateSkipAvailable = false
pcall(function()
    CrateModule = require(ReplicatedStorage:WaitForChild("Modules"):WaitForChild("Game")
        :WaitForChild("CrateSystem"):WaitForChild("Crate"))
    CrateSkipAvailable = type(CrateModule) == "table" and type(CrateModule.skip_spin) == "function"
        and type(CrateModule.spinning) == "table" and type(CrateModule.spinning.get) == "function"
end)
pcall(function()
    CrateSkipData = require(ReplicatedStorage:WaitForChild("Modules"):WaitForChild("Core"):WaitForChild("Data"))
end)
local CrateSkipSection = Tabs.General
CrateSkipSection:AddParagraph({
    Title = "สถานะข้ามแอนิเมชัน",
    Content = CrateSkipAvailable
        and "พบฟังก์ชันของเกมแล้ว แต่จะข้ามได้เมื่อบัญชีมีสิทธิ์ Skip Crate Animation ตามระบบเกม"
        or "ไม่พบ API ข้ามแอนิเมชันในเวอร์ชันเกมนี้",
})
local CrateSkipToggle = CrateSkipSection:AddToggle("CrateAnimationSkip", {
    Title = "ข้ามแอนิเมชันสุ่มอัตโนมัติ (เมื่อมีสิทธิ์)",
    Description = "ใช้ระบบข้ามของเกมเมื่อบัญชีมีสิทธิ์เท่านั้น ไม่ข้ามข้อกำหนดการซื้อของเกม",
    Default = false,
})
CrateSkipToggle:OnChanged(function()
    CrateSkipConfig.Enabled = Options.CrateAnimationSkip.Value == true
    CrateSkipConfig.Warned = false
    if CrateSkipConfig.Enabled then
        local entitled = false
        pcall(function() entitled = CrateSkipData and CrateSkipData.can_skip_crate_animation == true end)
        if not entitled then
            notify("ข้ามแอนิเมชันสุ่มของ", "เกมกำหนดให้ต้องมีสิทธิ์ Skip Crate Animation ก่อน ฟังก์ชันนี้จะไม่ข้ามหน้าซื้อหรือปลดล็อกสิทธิ์ให้ฟรี", 7)
            CrateSkipConfig.Warned = true
        end
    end
end)
task.spawn(function()
    while not Fluent.Unloaded do
        local entitled = false
        pcall(function() entitled = CrateSkipData and CrateSkipData.can_skip_crate_animation == true end)
        if CrateSkipConfig.Enabled and CrateSkipAvailable and CrateModule and entitled then
            local ok, spinning = pcall(function() return CrateModule.spinning.get() end)
            if ok and spinning then
                local now = os.clock()
                if not CrateSkipConfig.LastSpinning and now - CrateSkipConfig.LastCall > 0.5 then
                    CrateSkipConfig.LastSpinning = true
                    CrateSkipConfig.LastCall = now
                    local skipOk, skipErr = pcall(CrateModule.skip_spin)
                    if not skipOk then warn("[IMS Crate Skip] skip_spin error:", skipErr) end
                end
            else
                CrateSkipConfig.LastSpinning = false
            end
        else
            CrateSkipConfig.LastSpinning = false
        end
        task.wait(0.08)
    end
end)

end

-- สร้างหัวข้อย่อย ถ้า Fluent เวอร์ชันนี้ไม่รองรับ Section จะใช้ Tab เดิมแทน
local function newSection(tab, title)
    local ok, sec = pcall(function() return tab:AddSection(title) end)
    if ok and sec and sec.AddToggle then return sec end
    return tab
end

-- ========== แท็บ Main ==========
StatusPara = Tabs.Main:AddParagraph({
    Title = "สถานะ",
    Content = "เก็บสำเร็จแล้ว: 0 ชิ้น",
})

local SecCollect = newSection(Tabs.Main, "Auto-Collect")

local ModeDropdown = SecCollect:AddDropdown("CollectMode", {
    Title = "โหมด Auto-Collect",
    Description = "Pull = ดูดทุกชิ้นพร้อมกัน | Walk = เดินไปเก็บทีละชิ้นแล้วกลับที่เดิม",
    Values = { "Pull", "Walk" },
    Multi = false,
    Default = 1,
})

ModeDropdown:OnChanged(function()
    State.Mode = Options.CollectMode.Value
    State.Home = nil
    stopVacuum() -- เปลี่ยนโหมด: เคลียร์ของที่กำลังดูดอยู่ก่อน
end)

local CollectToggle = SecCollect:AddToggle("AutoCollect", {
    Title = "เปิด/ปิด Auto-Collect",
    Default = false,
})

CollectToggle:OnChanged(function()
    State.CollectEnabled = Options.AutoCollect.Value
    if not State.CollectEnabled then
        State.Home = nil
        stopVacuum()
        pcall(function()
            local hum, root = getHumanoid(), getRoot()
            if hum and root then hum:MoveTo(root.Position) end -- หยุดเดินทันที
        end)
    end
end)

local CollectKey = SecCollect:AddKeybind("CollectKey", {
    Title = "ปุ่มลัดเปิด/ปิด Auto-Collect",
    Mode = "Toggle",
    Default = "RightAlt",
    Callback = function() end,
    ChangedCallback = function() end,
})

CollectKey:OnClick(function()
    Options.AutoCollect:SetValue(not Options.AutoCollect.Value)
end)

local SecESP = newSection(Tabs.Main, "ESP")

local ESPToggle = SecESP:AddToggle("ESPToggle", {
    Title = "เปิด/ปิด ESP ไอเทมที่ตกพื้น",
    Description = "แสดง ชื่อ | ความแรร์ | ระยะ เหนือไอเทม",
    Default = false,
})

ESPToggle:OnChanged(function()
    State.ESPEnabled = Options.ESPToggle.Value
    if not State.ESPEnabled then clearAllESP() end
end)

local SecTools = newSection(Tabs.Main, "เครื่องมือ")

SecTools:AddButton({
    Title = "Debug",
    Description = "ดูว่าสคริปต์เห็นไอเทมอะไรบ้าง (ผลออกที่ Console F9 ด้วย)",
    Callback = function()
        local ok, err = pcall(runDebug)
        if not ok then warn("[ItemSystem] Debug error:", err) end
    end,
})

-- ========== แท็บ Player ESP ==========
local SecPE = newSection(Tabs.PlayerESP, "ดูไอเทมในตัวผู้เล่น")

local PEToggle = SecPE:AddToggle("PlayerESP", {
    Title = "เปิด/ปิด Player ESP",
    Description = "แสดงชื่อไอเทมของผู้เล่นคนอื่น สีตามความแรร์ (ทอง = Legendary, ม่วง = Epic)",
    Default = false,
})

PEToggle:OnChanged(function()
    State.PlayerESPEnabled = Options.PlayerESP.Value
    if not State.PlayerESPEnabled then clearAllPlayerESP() end
end)

local PERange = SecPE:AddSlider("PERange", {
    Title = "ระยะแสดงผล",
    Description = "หน่วย studs (สูงสุด 5,000)",
    Default = 5000,
    Min = 50,
    Max = 5000,
    Rounding = 0,
    Callback = function() end,
})

PERange:OnChanged(function(value)
    State.PERange = value
end)

local PERarity = SecPE:AddDropdown("PERarityFilter", {
    Title = "แสดงเฉพาะความแรร์",
    Description = "ติ๊กเฉพาะระดับที่อยากเห็น (เช่น ติ๊กแค่ Epic, Legendary, Omega)",
    Values = { "Common", "Uncommon", "Rare", "Epic", "Legendary", "Omega" },
    Multi = true,
    Default = { "Common", "Uncommon", "Rare", "Epic", "Legendary", "Omega" },
})

PERarity:OnChanged(function(value)
    local set = {}
    for name, on in pairs(value or {}) do
        if on then set[string.lower(tostring(name))] = true end
    end
    State.PERarityFilter = set
end)

local SecPEStyle = newSection(Tabs.PlayerESP, "หน้าตา")

local PEName = SecPEStyle:AddToggle("PEShowName", {
    Title = "แสดงชื่อผู้เล่น + ระยะ",
    Default = true,
})

PEName:OnChanged(function()
    State.PEShowName = Options.PEShowName.Value
end)

local PEHead = SecPEStyle:AddToggle("PEHeaderByRarity", {
    Title = "สีชื่อผู้เล่นตามไอเทมที่แรร์สุดในตัว",
    Default = true,
})

PEHead:OnChanged(function()
    State.PEHeaderByRarity = Options.PEHeaderByRarity.Value
end)

local PEHeld = SecPEStyle:AddToggle("PEShowHeld", {
    Title = "ทำเครื่องหมาย ▶ ที่ไอเทมที่กำลังถือ",
    Default = true,
})

PEHeld:OnChanged(function()
    State.PEShowHeld = Options.PEShowHeld.Value
end)

local PELines = SecPEStyle:AddSlider("PEMaxLines", {
    Title = "จำนวนไอเทมสูงสุดต่อคน",
    Description = "แสดงของแรร์สุดก่อน ที่เกินจะย่อเป็น +N",
    Default = 10,
    Min = 1,
    Max = 25,
    Rounding = 0,
    Callback = function() end,
})

PELines:OnChanged(function(value)
    State.PEMaxLines = value
end)

local PESize = SecPEStyle:AddSlider("PETextSize", {
    Title = "ขนาดตัวอักษร",
    Default = 12,
    Min = 9,
    Max = 22,
    Rounding = 0,
    Callback = function() end,
})

PESize:OnChanged(function(value)
    State.PETextSize = value
end)

-- ========== แท็บ Combat ==========
local SecCombat = newSection(Tabs.Combat, "ปืน + สเตมิน่า")

local CombatPara = Tabs.Combat:AddParagraph({
    Title = "สถานะระบบเกม",
    Content = "กำลังตรวจจับ... (กดปุ่มตรวจจับเพื่ออัปเดต)",
})

local function refreshDetect(showNotify)
    local info = detectSystems()
    local text = describeSystems(info)
    setParagraph(CombatPara, text)
    print("[ItemSystem][Combat] " .. text:gsub("\n", " | "))
    if showNotify then notify("ตรวจจับระบบเกม", text, 8) end
    return info
end

SecCombat:AddButton({
    Title = "ตรวจจับระบบเกม",
    Description = "เช็กว่าเกม/แมพนี้มีระบบสเตมิน่าและแรงสะท้อนที่สคริปต์รองรับหรือไม่",
    Callback = function() refreshDetect(true) end,
})

local RecoilToggle = SecCombat:AddToggle("NoRecoil", {
    Title = "ปืนไม่มีแรงสะท้อน (จอไม่สั่น)",
    Description = "ตั้งค่า Recoil ของปืนเป็น 0 ที่ฝั่งเครื่องเรา ปิดแล้วคืนค่าเดิมให้",
    Default = false,
})

RecoilToggle:OnChanged(function()
    State.NoRecoil = Options.NoRecoil.Value
    if State.NoRecoil then
        local info = detectSystems()
        if not info.Camera and info.Guns == 0 then
            notify("ไม่มีแรงสะท้อน", "ยังไม่พบปืนในตัว ลองหยิบปืนมาไว้ในกระเป๋าแล้วสคริปต์จะจับให้เอง", 6)
        end
        pcall(bindGunContainers)
        pcall(scanGuns)
    else
        restoreGuns()
    end
end)

local StaminaToggle = Tabs.General:AddToggle("InfStamina", {
    Title = "สเตมิน่าไม่จำกัด",
    Description = "ป้องกันการหักสเตมิน่าฝั่งไคลเอนต์และเติมเต็มต่อเนื่อง; หากเซิร์ฟเวอร์ตรวจเองอาจยังมีข้อจำกัด",
    Default = false,
})

StaminaToggle:OnChanged(function()
    State.InfStamina = Options.InfStamina.Value
    if State.InfStamina then
        local info = detectSystems()
        if not info.Sprint then
            notify("สเตมิน่า", "ไม่พบโมดูล Sprint ของเกม ฟีเจอร์นี้อาจใช้ไม่ได้ในแมพนี้", 6)
        end
    end
end)

-- [v18] แก้ต้นเหตุที่แถบฟื้นทันทีแต่ sprint ถูกปิดก่อนลูป 0.1s จะทำงาน:
-- ห่อ Sprint.consume_stamina ให้คืน true และไม่หักสเตมิน่าเมื่อเปิดโหมดนี้
-- ปิดโหมด/ปิดสคริปต์แล้วคืนฟังก์ชันเดิมกลับ
 do
    local hookedSprintModule = nil
    local originalConsumeStamina = nil
    local hookedConsumeStamina = nil

    local function restoreStaminaHook()
        if hookedSprintModule and hookedConsumeStamina then
            pcall(function()
                if hookedSprintModule.consume_stamina == hookedConsumeStamina then
                    hookedSprintModule.consume_stamina = originalConsumeStamina
                end
            end)
        end
        hookedSprintModule = nil
        originalConsumeStamina = nil
        hookedConsumeStamina = nil
    end

    task.spawn(function()
        while not Fluent.Unloaded do
            local sprint = loadGameModule("Sprint")
            if State.InfStamina and sprint and type(sprint.consume_stamina) == "function"
                and type(sprint.sprint_bar) == "table" and type(sprint.sprint_bar.set) == "function" then
                if hookedSprintModule ~= sprint or sprint.consume_stamina ~= hookedConsumeStamina then
                    restoreStaminaHook()
                    hookedSprintModule = sprint
                    originalConsumeStamina = sprint.consume_stamina
                    hookedConsumeStamina = function(amount)
                        if State.InfStamina then
                            pcall(function() hookedSprintModule.sprint_bar.set(1) end)
                            return true
                        end
                        if originalConsumeStamina then
                            return originalConsumeStamina(amount)
                        end
                        return false
                    end
                    pcall(function() sprint.consume_stamina = hookedConsumeStamina end)
                end
                pcall(function() sprint.sprint_bar.set(1) end)
            elseif not State.InfStamina then
                restoreStaminaHook()
            end
            task.wait(0.05)
        end
        restoreStaminaHook()
    end)
end

task.spawn(function()
    task.wait(3)
    pcall(refreshDetect, false)
end)

do -- [FIXED v16.1] แยก scope Aimbot ป้องกัน Lua/Luau เกินขีดจำกัด local 200 ตัว
-- ========== แท็บ Aimbot (กล้องล็อกเป้าในวง FOV) ==========
-- [FIXED] รุ่นนี้ทำ Camera Aim Assist แบบปกติเท่านั้น: เลือกเป้าจากจอและหันกล้องให้
-- ไม่เปลี่ยน Remote/ทิศทางกระสุนของเซิร์ฟเวอร์ (ไม่ใช่ Silent Aim)
local AimAssist = {
    Enabled = false,
    FOV = 140,             -- รัศมีวงในหน่วย pixels
    Part = "Head",        -- Head หรือ Body
    Smoothness = 0.30,     -- 0.05 = นุ่ม / 1 = หันกล้องเร็วที่สุด
    ShowFOV = true,
    TargetPlayer = nil,
    TargetName = "ยังไม่มีเป้า",
}

local AimGui, AimCircle, AimCircleStroke
local aimGuiOk = pcall(function()
    AimGui = Ext.makeGui("IMS_Aimbot_FOV_v16", 940)
    AimGui.IgnoreGuiInset = true
    AimGui.ResetOnSpawn = false
    AimCircle = Instance.new("Frame")
    AimCircle.Name = "FOVCircle"
    AimCircle.AnchorPoint = Vector2.new(0.5, 0.5)
    AimCircle.Position = UDim2.fromScale(0.5, 0.5)
    AimCircle.Size = UDim2.fromOffset(AimAssist.FOV * 2, AimAssist.FOV * 2)
    AimCircle.BackgroundTransparency = 1
    AimCircle.BorderSizePixel = 0
    AimCircle.Visible = false
    AimCircle.Parent = AimGui
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = AimCircle
    AimCircleStroke = Instance.new("UIStroke")
    AimCircleStroke.Thickness = 1.5
    AimCircleStroke.Color = Color3.fromRGB(70, 255, 130)
    AimCircleStroke.Transparency = 0.08
    AimCircleStroke.Parent = AimCircle
end)
if not aimGuiOk then warn("[IMS Aimbot] FOV circle UI failed to initialize") end

local AimSection = newSection(Tabs.Aimbot, "Aimbot / FOV")
local AimStatusPara = Tabs.Aimbot:AddParagraph({
    Title = "สถานะเป้าหมาย",
    Content = "ปิดอยู่ — เปิด Aimbot เพื่อเริ่มเลือกเป้าภายในวง FOV",
})

local AimOnToggle = AimSection:AddToggle("AimbotEnabled", {
    Title = "เปิด/ปิด Aimbot (Camera Lock)",
    Description = "เลือกผู้เล่นที่อยู่ในวง FOV แล้วหันกล้องตามเป้าอัตโนมัติ",
    Default = false,
})
AimOnToggle:OnChanged(function()
    AimAssist.Enabled = Options.AimbotEnabled.Value == true
    if not AimAssist.Enabled then
        AimAssist.TargetPlayer = nil
        AimAssist.TargetName = "ปิดอยู่"
    end
    if AimCircle then AimCircle.Visible = AimAssist.Enabled and AimAssist.ShowFOV end
end)

local AimFOVSlider = AimSection:AddSlider("AimbotFOV", {
    Title = "ขนาดวง FOV (รัศมี pixels)",
    Description = "ผู้เล่นอยู่นอกวงนี้จะไม่ถูกล็อก",
    Default = AimAssist.FOV,
    Min = 30,
    Max = 500,
    Rounding = 0,
    Callback = function() end,
})
AimFOVSlider:OnChanged(function(value)
    AimAssist.FOV = math.clamp(tonumber(value) or 140, 30, 500)
    if AimCircle then AimCircle.Size = UDim2.fromOffset(AimAssist.FOV * 2, AimAssist.FOV * 2) end
end)

local AimPartDropdown = AimSection:AddDropdown("AimbotTargetPart", {
    Title = "ตำแหน่งล็อกเป้า",
    Description = "Head = ล็อกหัว | Body = ล็อกลำตัว",
    Values = { "Head", "Body" },
    Multi = false,
    Default = 1,
})
AimPartDropdown:OnChanged(function(value)
    local selected = Options.AimbotTargetPart.Value
    if type(selected) == "string" then AimAssist.Part = selected
    elseif type(value) == "string" then AimAssist.Part = value end
    AimAssist.TargetPlayer = nil
end)

-- [v18] Friend/exclusion picker: checkbox rows for players currently in this server.
-- ไม่ใช้ searchable dropdown เพราะ Fluent บางเวอร์ชันทำให้ต้องพิมพ์ชื่อก่อนเลือก
local AimExcludedUserIds = {}
do
    local AimFriendsGui = Instance.new("ScreenGui")
    AimFriendsGui.Name = "IMS_AimFriendPicker"
    AimFriendsGui.ResetOnSpawn = false
    AimFriendsGui.DisplayOrder = 99999
    AimFriendsGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    AimFriendsGui.Enabled = false

    local guiParent = LocalPlayer:FindFirstChildOfClass("PlayerGui") or LocalPlayer:WaitForChild("PlayerGui")
    local oldPicker = guiParent:FindFirstChild("IMS_AimFriendPicker")
    if oldPicker then pcall(function() oldPicker:Destroy() end) end
    AimFriendsGui.Parent = guiParent

    local panel = Instance.new("Frame")
    panel.Name = "Panel"
    panel.AnchorPoint = Vector2.new(0.5, 0.5)
    panel.Position = UDim2.new(0.5, 0, 0.5, 0)
    panel.Size = UDim2.fromOffset(340, 390)
    panel.BackgroundColor3 = Color3.fromRGB(24, 26, 32)
    panel.BorderSizePixel = 0
    panel.Parent = AimFriendsGui
    Instance.new("UICorner", panel).CornerRadius = UDim.new(0, 10)

    local header = Instance.new("TextLabel")
    header.BackgroundTransparency = 1
    header.Position = UDim2.fromOffset(12, 8)
    header.Size = UDim2.new(1, -58, 0, 30)
    header.Font = Enum.Font.GothamBold
    header.Text = "Friend List / ผู้เล่นที่ไม่ล็อกเป้า"
    header.TextColor3 = Color3.fromRGB(245, 245, 250)
    header.TextSize = 14
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Parent = panel

    local closeButton = Instance.new("TextButton")
    closeButton.Position = UDim2.new(1, -38, 0, 7)
    closeButton.Size = UDim2.fromOffset(28, 28)
    closeButton.BackgroundColor3 = Color3.fromRGB(60, 63, 74)
    closeButton.Text = "X"
    closeButton.Font = Enum.Font.GothamBold
    closeButton.TextSize = 13
    closeButton.TextColor3 = Color3.new(1, 1, 1)
    closeButton.Parent = panel
    Instance.new("UICorner", closeButton).CornerRadius = UDim.new(0, 6)
    closeButton.MouseButton1Click:Connect(function() AimFriendsGui.Enabled = false end)

    local hint = Instance.new("TextLabel")
    hint.BackgroundTransparency = 1
    hint.Position = UDim2.fromOffset(12, 39)
    hint.Size = UDim2.new(1, -24, 0, 27)
    hint.Font = Enum.Font.Gotham
    hint.Text = "คลิกชื่อเพื่อเลือก/ยกเลิก • รายชื่อจากเซิร์ฟเวอร์ปัจจุบัน"
    hint.TextColor3 = Color3.fromRGB(180, 185, 197)
    hint.TextSize = 10
    hint.TextXAlignment = Enum.TextXAlignment.Left
    hint.Parent = panel

    local scrolling = Instance.new("ScrollingFrame")
    scrolling.Name = "PlayerList"
    scrolling.Position = UDim2.fromOffset(12, 70)
    scrolling.Size = UDim2.new(1, -24, 1, -118)
    scrolling.BackgroundColor3 = Color3.fromRGB(32, 35, 43)
    scrolling.BorderSizePixel = 0
    scrolling.ScrollBarThickness = 5
    scrolling.CanvasSize = UDim2.new(0, 0, 0, 0)
    scrolling.Parent = panel
    Instance.new("UICorner", scrolling).CornerRadius = UDim.new(0, 7)
    local listLayout = Instance.new("UIListLayout")
    listLayout.Padding = UDim.new(0, 4)
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder
    listLayout.Parent = scrolling
    listLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        scrolling.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 8)
    end)

    local refreshButton = Instance.new("TextButton")
    refreshButton.Position = UDim2.new(0, 12, 1, -39)
    refreshButton.Size = UDim2.new(1, -24, 0, 28)
    refreshButton.BackgroundColor3 = Color3.fromRGB(56, 90, 140)
    refreshButton.Text = "รีเฟรชรายชื่อ"
    refreshButton.Font = Enum.Font.GothamSemibold
    refreshButton.TextSize = 12
    refreshButton.TextColor3 = Color3.new(1, 1, 1)
    refreshButton.Parent = panel
    Instance.new("UICorner", refreshButton).CornerRadius = UDim.new(0, 6)

    local function refreshAimFriendRows()
        for _, child in ipairs(scrolling:GetChildren()) do
            if child:IsA("GuiObject") and child ~= listLayout then child:Destroy() end
        end
        local currentPlayers = {}
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then table.insert(currentPlayers, player) end
        end
        table.sort(currentPlayers, function(a, b)
            return string.lower(a.Name) < string.lower(b.Name)
        end)
        if #currentPlayers == 0 then
            local emptyLabel = Instance.new("TextLabel")
            emptyLabel.Name = "Empty"
            emptyLabel.LayoutOrder = 1
            emptyLabel.Size = UDim2.new(1, -8, 0, 34)
            emptyLabel.BackgroundTransparency = 1
            emptyLabel.Text = "ยังไม่มีผู้เล่นคนอื่นในเซิร์ฟเวอร์"
            emptyLabel.Font = Enum.Font.Gotham
            emptyLabel.TextSize = 11
            emptyLabel.TextColor3 = Color3.fromRGB(210, 210, 215)
            emptyLabel.Parent = scrolling
            return
        end
        for index, player in ipairs(currentPlayers) do
            local selected = AimExcludedUserIds[player.UserId] == true
            local row = Instance.new("TextButton")
            row.Name = "Player_" .. tostring(player.UserId)
            row.LayoutOrder = index
            row.Size = UDim2.new(1, -8, 0, 34)
            row.BackgroundColor3 = selected and Color3.fromRGB(42, 100, 75) or Color3.fromRGB(46, 49, 58)
            row.BorderSizePixel = 0
            row.AutoButtonColor = true
            row.Font = Enum.Font.Gotham
            row.TextSize = 11
            row.TextColor3 = Color3.fromRGB(245, 245, 250)
            row.TextXAlignment = Enum.TextXAlignment.Left
            row.Text = (selected and "  [X]  " or "  [ ]  ") .. player.Name .. "  •  " .. tostring(player.UserId)
            row.Parent = scrolling
            Instance.new("UICorner", row).CornerRadius = UDim.new(0, 5)
            row.MouseButton1Click:Connect(function()
                if AimExcludedUserIds[player.UserId] then
                    AimExcludedUserIds[player.UserId] = nil
                else
                    AimExcludedUserIds[player.UserId] = true
                end
                AimAssist.TargetPlayer = nil
                refreshAimFriendRows()
            end)
        end
    end

    refreshButton.MouseButton1Click:Connect(refreshAimFriendRows)
    local AimOpenFriendsButton = AimSection:AddButton({
        Title = "เปิด Friend List (ติ๊กชื่อ ไม่ต้องพิมพ์)",
        Description = "เลือกผู้เล่นในเซิร์ฟเวอร์ที่จะให้ Aim Assist ข้าม",
        Callback = function()
            refreshAimFriendRows()
            AimFriendsGui.Enabled = true
        end,
    })
    Players.PlayerAdded:Connect(function()
        task.delay(0.4, function()
            if AimFriendsGui.Parent then refreshAimFriendRows() end
        end)
    end)
    Players.PlayerRemoving:Connect(function(player)
        -- เก็บ UserId ที่เลือกไว้ เพื่อให้ยังถูกยกเว้นถ้าผู้เล่นคนนั้นกลับเข้าเซิร์ฟเวอร์
        AimAssist.TargetPlayer = nil
        task.delay(0.1, function()
            if AimFriendsGui.Parent then refreshAimFriendRows() end
        end)
    end)
    task.spawn(function()
        while not Fluent.Unloaded do task.wait(0.5) end
        pcall(function() AimFriendsGui:Destroy() end)
    end)
end

local AimSmoothSlider = AimSection:AddSlider("AimbotSmoothness", {
    Title = "ความเร็วการล็อกกล้อง",
    Description = "ค่าต่ำจะนุ่มขึ้น; ค่าสูงจะหันกล้องเร็วขึ้น",
    Default = AimAssist.Smoothness,
    Min = 0.05,
    Max = 1,
    Rounding = 2,
    Callback = function() end,
})
AimSmoothSlider:OnChanged(function(value)
    AimAssist.Smoothness = math.clamp(tonumber(value) or 0.30, 0.05, 1)
end)

local AimShowToggle = AimSection:AddToggle("AimbotShowFOV", {
    Title = "แสดงวง FOV",
    Default = true,
})
AimShowToggle:OnChanged(function()
    AimAssist.ShowFOV = Options.AimbotShowFOV.Value == true
    if AimCircle then AimCircle.Visible = AimAssist.Enabled and AimAssist.ShowFOV end
end)

AimSection:AddParagraph({
    Title = "หมายเหตุ",
    Content = "ล็อกผ่านกล้องเท่านั้น ไม่เปลี่ยนทิศทางกระสุนฝั่งเซิร์ฟเวอร์ และเป้าจะต้องอยู่ในวง FOV",
})

local function getAimPart(player)
    if not player or player == LocalPlayer then return nil end
    local char = player.Character
    if not char then return nil end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return nil end
    local part
    if AimAssist.Part == "Head" then
        part = char:FindFirstChild("Head")
    else
        part = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso")
            or char:FindFirstChild("HumanoidRootPart")
    end
    if part and part:IsA("BasePart") then return part end
    return nil
end

local function getScreenTarget(player, camera, center)
    -- [FIXED v17] ผู้เล่นที่ติ๊กในรายการยกเว้นจะไม่เป็นเป้าหมาย
    if player and AimExcludedUserIds[player.UserId] then return nil end
    local part = getAimPart(player)
    if not part then return nil end
    local point, onScreen = camera:WorldToViewportPoint(part.Position)
    if not onScreen or point.Z <= 0 then return nil end
    local distance = (Vector2.new(point.X, point.Y) - center).Magnitude
    if distance > AimAssist.FOV then return nil end
    return part, distance
end

local function chooseAimTarget(camera, center)
    -- [FIXED] Sticky lock: คงเป้าเดิมตราบใดที่ยังมีชีวิตและอยู่ในวง FOV ลดการสลับเป้าไปมา
    local current = AimAssist.TargetPlayer
    if current then
        local part = getScreenTarget(current, camera, center)
        if part then return current, part end
    end
    local bestPlayer, bestPart, bestDistance = nil, nil, math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local part, distance = getScreenTarget(player, camera, center)
            if part and distance and distance < bestDistance then
                bestPlayer, bestPart, bestDistance = player, part, distance
            end
        end
    end
    AimAssist.TargetPlayer = bestPlayer
    return bestPlayer, bestPart
end

local AIM_BIND_NAME = "IMS_Aimbot_CameraLock_v16"
pcall(function() RunService:UnbindFromRenderStep(AIM_BIND_NAME) end)
local lastAimScan = 0
local cachedAimPart = nil
local cachedAimPlayer = nil
local AimLastViewport = Vector2.new(0, 0)

local function updateAimAssist()
    local camera = Workspace.CurrentCamera
    if not camera then
        AimAssist.TargetPlayer, cachedAimPlayer, cachedAimPart = nil, nil, nil
        if AimCircle then AimCircle.Visible = false end
        return
    end
    local viewport = camera.ViewportSize
    local center = Vector2.new(viewport.X * 0.5, viewport.Y * 0.5)

    if AimCircle then
        if AimLastViewport ~= viewport then
            AimCircle.Position = UDim2.fromOffset(center.X, center.Y)
            AimCircle.Size = UDim2.fromOffset(AimAssist.FOV * 2, AimAssist.FOV * 2)
            AimLastViewport = viewport
        end
        AimCircle.Visible = AimAssist.Enabled and AimAssist.ShowFOV
    end

    if not AimAssist.Enabled then
        AimAssist.TargetPlayer, cachedAimPlayer, cachedAimPart = nil, nil, nil
        return
    end

    local now = os.clock()
    if now - lastAimScan >= 0.08 or not cachedAimPart or not cachedAimPart.Parent then
        lastAimScan = now
        cachedAimPlayer, cachedAimPart = chooseAimTarget(camera, center)
    else
        -- ตรวจซ้ำว่าเป้ายังอยู่ในวงทุกเฟรม แม้ไม่ได้สแกนรายชื่อผู้เล่นใหม่
        local validPart = cachedAimPlayer and getScreenTarget(cachedAimPlayer, camera, center)
        if not validPart then
            AimAssist.TargetPlayer, cachedAimPlayer, cachedAimPart = nil, nil, nil
        else
            cachedAimPart = validPart
        end
    end

    if cachedAimPlayer and cachedAimPart and cachedAimPart.Parent then
        AimAssist.TargetName = tostring(cachedAimPlayer.DisplayName or cachedAimPlayer.Name)
        local goal = CFrame.lookAt(camera.CFrame.Position, cachedAimPart.Position)
        local alpha = math.clamp(AimAssist.Smoothness, 0.05, 1)
        camera.CFrame = camera.CFrame:Lerp(goal, alpha)
    else
        AimAssist.TargetName = "ไม่มีเป้าในวง FOV"
    end
end

local aimBindOk, aimBindErr = pcall(function()
    RunService:BindToRenderStep(AIM_BIND_NAME, Enum.RenderPriority.Camera.Value + 1, function()
        local ok, err = pcall(updateAimAssist)
        if not ok then warn("[IMS Aimbot] update error:", err) end
    end)
end)
if not aimBindOk then warn("[IMS Aimbot] render binding failed:", tostring(aimBindErr)) end

task.spawn(function()
    while not Fluent.Unloaded do
        local text = AimAssist.Enabled and ("เปิดอยู่ | เป้าหมาย: " .. tostring(AimAssist.TargetName)
            .. " | จุดเล็ง: " .. AimAssist.Part .. " | FOV radius: " .. tostring(AimAssist.FOV))
            or "ปิดอยู่ — เปิด Aimbot เพื่อเริ่มเลือกเป้าภายในวง FOV"
        pcall(setParagraph, AimStatusPara, text)
        task.wait(0.25)
    end
    pcall(function() RunService:UnbindFromRenderStep(AIM_BIND_NAME) end)
    if AimGui then pcall(function() AimGui:Destroy() end) end
end)

end -- end Aimbot local scope

-- ========== แท็บ Movement ==========
do
    local B = Ext.Burrow
    local Sec = newSection(Tabs.Movement, "มุดดิน (Burrow)")

    Tabs.Movement:AddParagraph({
        Title = "คำเตือน",
        Content = "ตำแหน่งตัวละครจะอยู่ใต้พื้นจริง เซิร์ฟเวอร์เห็นตำแหน่งนี้ เกมนี้มีระบบตรวจจับ อาจโดนเตะ/แบนได้ ใช้ด้วยความเสี่ยงของตัวเอง",
    })

    local On = Sec:AddToggle("BurrowEnabled", {
        Title = "เปิดระบบมุดดิน",
        Description = "เปิดแล้ว: คอมกดปุ่มที่ตั้งไว้ (ค่าเริ่มต้น Z) / มือถือกดปุ่ม Snap เพื่อมุดลง กดซ้ำเพื่อขึ้น",
        Default = false,
    })
    On:OnChanged(function() B.setEnabled(Options.BurrowEnabled.Value) end)

    local Key = Sec:AddKeybind("BurrowKey", {
        Title = "ปุ่มมุดดิน (คอม)",
        Mode = "Toggle",
        Default = "Z",
        Callback = function() end,
        ChangedCallback = function() end,
    })
    pcall(function()
        Key:OnChanged(function() B.setKey(Options.BurrowKey.Value) end)
    end)
    pcall(function() B.setKey(Options.BurrowKey.Value) end)

    local Depth = Sec:AddSlider("BurrowDepth", {
        Title = "ระยะที่มุดลง",
        Description = "หน่วย studs (ค่าเริ่มต้น 10)",
        Default = 10,
        Min = 3,
        Max = 100,
        Rounding = 0,
        Callback = function() end,
    })
    Depth:OnChanged(function(value) B.Depth = value end)

    local ShowBtn = Sec:AddToggle("BurrowButton", {
        Title = "แสดงปุ่ม Snap บนหน้าจอ",
        Description = "สำหรับมือถือ ลากย้ายตำแหน่งปุ่มได้",
        Default = B.ShowButton,
    })
    ShowBtn:OnChanged(function() B.setShowButton(Options.BurrowButton.Value) end)

    Sec:AddButton({
        Title = "Snap ตอนนี้",
        Description = "ทดสอบมุดลง/ขึ้นจากเมนู",
        Callback = function()
            if not B.Enabled then
                notify("มุดดิน", "กรุณาเปิดระบบมุดดินก่อน", 3)
                return
            end
            B.toggle()
        end,
    })
end

-- [FIXED] นำแท็บ Fish ออกเพื่อให้เหลือเมนูหลักด้าน Combat/ESP/Aimbot

-- ========== แท็บ General ==========
local SecPull = newSection(Tabs.General, "การเก็บของ")

local RangeSlider = SecPull:AddSlider("Range", {
    Title = "ระยะตรวจจับ",
    Description = "หน่วย studs (ค่าเริ่มต้น 20)",
    Default = 20,
    Min = 5,
    Max = 40,
    Rounding = 0,
    Callback = function() end,
})

RangeSlider:OnChanged(function(value)
    State.Range = value
end)

local SpeedSlider = SecPull:AddSlider("PullSpeed", {
    Title = "ความเร็วดูด (โหมด Pull)",
    Description = "1.00 = ดูดทันทีทั้งหมด, ต่ำกว่านี้ = ค่อย ๆ ดูด",
    Default = 1,
    Min = 0.2,
    Max = 1,
    Rounding = 2,
    Callback = function() end,
})

SpeedSlider:OnChanged(function(value)
    State.PullSpeed = value
end)

local MoveToggle = SecPull:AddToggle("PullMove", {
    Title = "ย้ายของเข้าหาตัวด้วย (ทดลอง)",
    Description = "ปิดไว้ = สั่งเก็บทุกชิ้นพร้อมกันโดยไม่ย้ายของ (แนะนำ เพราะเกมคุมตำแหน่งของเอง)",
    Default = false,
})

MoveToggle:OnChanged(function()
    State.PullMove = Options.PullMove.Value
    stopVacuum()
end)

local NotifyToggle = SecPull:AddToggle("NotifyCollect", {
    Title = "แจ้งเตือนเมื่อเก็บสำเร็จ",
    Description = "รวมเป็นชุด ไม่แจ้งทีละชิ้นให้จอรก",
    Default = true,
})

NotifyToggle:OnChanged(function()
    State.NotifyCollect = Options.NotifyCollect.Value
end)

local SecESPStyle = newSection(Tabs.General, "หน้าตา ESP")

local ESPColorMode = SecESPStyle:AddDropdown("ESPColorMode", {
    Title = "สีป้าย ESP",
    Values = { "Rarity", "Custom" },
    Multi = false,
    Default = 1,
})

ESPColorMode:OnChanged(function()
    State.ESPColorMode = Options.ESPColorMode.Value
end)

local ESPColorPicker = SecESPStyle:AddColorpicker("ESPColor", {
    Title = "สีกำหนดเอง (ใช้เมื่อเลือก Custom)",
    Default = Color3.fromRGB(255, 255, 255),
})

ESPColorPicker:OnChanged(function()
    State.ESPColor = ESPColorPicker.Value
end)

local ESPSize = SecESPStyle:AddSlider("ESPTextSize", {
    Title = "ขนาดตัวอักษร ESP",
    Default = 14,
    Min = 10,
    Max = 24,
    Rounding = 0,
    Callback = function() end,
})

ESPSize:OnChanged(function(value)
    State.ESPTextSize = value
end)

local SecGui = newSection(Tabs.General, "GUI")

local ScaleSlider = SecGui:AddSlider("GuiScale", {
    Title = "ขนาด GUI",
    Description = "หน่วย % (ค่าเริ่มต้น 100)",
    Default = 100,
    Min = 50,
    Max = 150,
    Rounding = 0,
    Callback = function() end,
})

ScaleSlider:OnChanged(function(value)
    pcall(setGuiScale, value)
end)

local SecPerf = newSection(Tabs.General, "ประสิทธิภาพ (เครื่องอ่อน)")

local PerfDropdown = SecPerf:AddDropdown("PerfMode", {
    Title = "โหมดลดกราฟิก (เพิ่ม FPS)",
    Description = "Ultra Low ภาพกากมาก แต่ลื่นสุด | เลือก Off เพื่อคืนค่าเดิม",
    Values = { "Off", "Low", "Ultra Low" },
    Multi = false,
    Default = 1,
})

PerfDropdown:OnChanged(function()
    local value = Options.PerfMode.Value
    local level = (value == "Low" and 1) or (value == "Ultra Low" and 2) or 0
    local ok, err = pcall(setPerfLevel, level)
    if not ok then warn("[ItemSystem] setPerfLevel error:", err) end
end)

FpsPara = SecPerf:AddParagraph({
    Title = "FPS",
    Content = "...",
})

FpsLockPara = SecPerf:AddParagraph({
    Title = "FPS Lock",
    Content = FpsLock.Supported and "FPS Lock: ปิด" or "FPS Lock: executor ไม่รองรับ setfpscap",
})

local FpsCapSlider = SecPerf:AddSlider("FpsCap", {
    Title = "ล็อก FPS ที่",
    Description = "เลือกเพดาน FPS ตั้งแต่ 15-240",
    Default = 60,
    Min = 15,
    Max = 240,
    Rounding = 0,
    Callback = function() end,
})
FpsCapSlider:OnChanged(function(value)
    FpsLock.Cap = math.floor(tonumber(value) or 60)
    if FpsLock.Enabled then
        pcall(setFpsLock, true, FpsLock.Cap)
    end
end)

local FpsLockToggle = SecPerf:AddToggle("FpsLock", {
    Title = "ล็อก FPS",
    Description = "ใช้ setfpscap เพื่อกำหนดเพดาน FPS ของ executor",
    Default = false,
})
FpsLockToggle:OnChanged(function()
    pcall(setFpsLock, Options.FpsLock.Value, Options.FpsCap.Value)
end)

local SecDiscord = newSection(Tabs.General, "Discord Webhook")

local DiscordWebhookInput = SecDiscord:AddInput("DiscordWebhookURL", {
    Title = "Discord Webhook URL",
    Description = "ใส่ webhook ของเซิร์ฟเวอร์ Discord ที่คุณเป็นเจ้าของ",
    Default = CONFIG.DISCORD_WEBHOOK_URL,
    Placeholder = "https://discord.com/api/webhooks/...",
    Numeric = false,
    Finished = false,
})
DiscordWebhookInput:OnChanged(function(value)
    CONFIG.DISCORD_WEBHOOK_URL = tostring(value or "")
end)

local DiscordPingInput = SecDiscord:AddInput("DiscordPingID", {
    Title = "Discord User ID ที่จะให้แท็ก (ไม่ใส่ก็ได้)",
    Default = "",
    Placeholder = "เช่น 123456789012345678",
    Numeric = false,
    Finished = false,
})
DiscordPingInput:OnChanged(function(value)
    CONFIG.DISCORD_PING_ID = tostring(value or "")
end)

local DiscordToggle = SecDiscord:AddToggle("DeathWebhook", {
    Title = "แจ้งเตือนเมื่อเราตาย",
    Description = "ส่งสาเหตุ ตำแหน่ง ผู้เล่นใกล้ตัว และของในตัวตอนตาย ไป Discord",
    Default = true,
})
DiscordToggle:OnChanged(function()
    State.DeathWebhook = Options.DeathWebhook.Value
    if State.DeathWebhook and tostring(CONFIG.DISCORD_WEBHOOK_URL or "") == "" then
        notify("Discord Webhook", "ยังไม่ได้ใส่ Webhook URL", 6)
    end
end)

SecDiscord:AddButton({
    Title = "ทดสอบ Discord Webhook",
    Description = "ส่งข้อความทดสอบ 1 ครั้ง",
    Callback = function()
        task.spawn(function()
            local ok, err = sendDiscordWebhook("ทดสอบ Item Management System", "Webhook ทำงานแล้ว", {
                { name = "Player", value = tostring(LocalPlayer.Name), inline = true },
                { name = "PlaceId", value = tostring(game.PlaceId), inline = true },
                { name = "JobId", value = tostring(game.JobId ~= "" and game.JobId or "Private/Studio"), inline = false },
            }, 3447003)
            notify("Discord", ok and "ส่งสำเร็จ ดูในห้อง Discord ได้เลย" or ("ส่งไม่สำเร็จ: " .. tostring(err)), 7)
            if not ok then warn("[ItemSystem][Discord] test failed:", err) end
        end)
    end,
})

do
    local Sec = newSection(Tabs.General, "ภาษา / Language")
    local Dd = Sec:AddDropdown("Language", {
        Title = "ภาษา (Language)",
        Values = { "ไทย", "English" },
        Multi = false,
        Default = (Ext.lang == "en") and "English" or "ไทย",
    })
    Dd:OnChanged(function(value)
        local code = (value == "English") and "en" or "th"
        if code == Ext.lang then return end
        Ext.setLang(code)
        notify("Language", "เปลี่ยนภาษาเรียบร้อย", 3)
    end)
end

do
    local Sec = newSection(Tabs.General, "อุปกรณ์ (Device)")
    local Dev = Sec:AddDropdown("Platform", {
        Title = "อุปกรณ์ (Device)",
        Values = { "โทรศัพท์", "คอม" },
        Multi = false,
        Default = (Ext.platform == "mobile") and "โทรศัพท์" or "คอม",
    })
    Dev:OnChanged(function(value)
        local plat = (value == "โทรศัพท์") and "mobile" or "pc"
        if plat == Ext.platform then return end
        Ext.platform = plat
        Ext.savePlatform()
        -- โทรศัพท์ = เปิดปุ่ม IMS + Snap, คอม = ซ่อน (ใช้ RightControl / Z)
        pcall(function() Options.MenuButton:SetValue(plat == "mobile") end)
        pcall(function() Options.BurrowButton:SetValue(plat == "mobile") end)
        notify("Device", "เปลี่ยนอุปกรณ์แล้ว ขนาดหน้าต่างจะปรับเมื่อรันสคริปต์ครั้งหน้า", 5)
    end)
end

local SecUnload = newSection(Tabs.General, "สคริปต์")

SecUnload:AddButton({
    Title = "ปิดสคริปต์ทั้งหมด",
    Description = "ปิด UI และคืนค่าทุกอย่างกลับเป็นปกติ (ESP, กราฟิก, การดูด)",
    Callback = function()
        pcall(function() Fluent:Destroy() end)
    end,
})

-- ========== แท็บ Settings (ธีม + คอนฟิกของ Fluent) ==========
pcall(function()
    if SaveManager and InterfaceManager then
        SaveManager:SetLibrary(Fluent)
        InterfaceManager:SetLibrary(Fluent)
        SaveManager:IgnoreThemeSettings()
        SaveManager:SetIgnoreIndexes({ "Language", "Platform" })
        InterfaceManager:SetFolder("ItemManagementSystem")
        SaveManager:SetFolder("ItemManagementSystem/config")
        InterfaceManager:BuildInterfaceSection(Tabs.Settings)
        SaveManager:BuildConfigSection(Tabs.Settings)
    end
end)

Window:SelectTab(1)

notify("Item Management System", "โหลดสำเร็จ กด RightControl เพื่อซ่อน/แสดงหน้าต่าง", 6)

pcall(function()
    if SaveManager then SaveManager:LoadAutoloadConfig() end
end)



-- ========== ปุ่มเมนูลอย (ซ่อน/แสดงหน้าต่างบนมือถือ โดยไม่ต้องใช้ RightControl) ==========
do
    local UserInputService = game:GetService("UserInputService")
    local menuGui, menuBtn

    -- ไอคอนปุ่ม: ฝังรูปไว้ในสคริปต์ (JPEG ขนาดเล็ก) เขียนเป็นไฟล์แล้วใช้ getcustomasset
    local ICON_B64 = [[
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
]]

    local function decodeBase64(data)
        local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
        local lookup = {}
        for i = 1, 64 do lookup[chars:sub(i, i)] = i - 1 end
        data = data:gsub("[^%w%+/]", "")
        local out = {}
        for i = 1, #data, 4 do
            local a = lookup[data:sub(i, i)]
            local b = lookup[data:sub(i + 1, i + 1)]
            local c = lookup[data:sub(i + 2, i + 2)]
            local d = lookup[data:sub(i + 3, i + 3)]
            local n = a * 262144 + b * 4096 + (c or 0) * 64 + (d or 0)
            table.insert(out, string.char(math.floor(n / 65536) % 256))
            if c then table.insert(out, string.char(math.floor(n / 256) % 256)) end
            if d then table.insert(out, string.char(n % 256)) end
        end
        return table.concat(out)
    end

    local function getIconAsset()
        local getAsset = getcustomasset or getsynasset
        if not (writefile and isfile and getAsset) then return nil end
        local path = "ItemManagementSystem/icon.jpg"
        local ok, asset = pcall(function()
            if makefolder and isfolder and not isfolder("ItemManagementSystem") then
                makefolder("ItemManagementSystem")
            end
            if not isfile(path) then writefile(path, decodeBase64(ICON_B64)) end
            return getAsset(path)
        end)
        if ok and asset and asset ~= "" then return asset end
        return nil
    end

    -- ซ่อน/แสดงหน้าต่างแบบชัดเจน (ไม่พึ่งสถานะสลับของ Fluent อย่างเดียว กันต้องกดซ้ำ)
    local function toggleWindow()
        local root
        pcall(function() root = Window.Root end)
        local before = root and root.Visible
        local ok = pcall(function() Window:Minimize() end)
        if root and before ~= nil then
            task.wait(0.05)
            if (not ok) or root.Visible == before then
                root.Visible = not before
                pcall(function() Window.Minimized = before end)
            end
        end
    end

    local okMenu = pcall(function()
        menuGui = Ext.makeGui("IMS_MenuButton", 100)

        local asset = getIconAsset()
        if asset then
            menuBtn = Instance.new("ImageButton")
            menuBtn.Image = asset
            menuBtn.ScaleType = Enum.ScaleType.Crop
        else
            menuBtn = Instance.new("TextButton")
            menuBtn.Text = "IMS"
            menuBtn.Font = Enum.Font.GothamBold
            menuBtn.TextSize = 15
            menuBtn.TextColor3 = Color3.new(1, 1, 1)
        end
        menuBtn.Size = UDim2.fromOffset(58, 58)
        menuBtn.Position = UDim2.new(0, 14, 0.3, 0)
        menuBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
        menuBtn.BackgroundTransparency = 0.05
        menuBtn.AutoButtonColor = false
        menuBtn.Active = true
        menuBtn.ClipsDescendants = true
        menuBtn.Parent = menuGui

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1, 0)
        corner.Parent = menuBtn
        local stroke = Instance.new("UIStroke")
        stroke.Color = Color3.fromRGB(120, 170, 255)
        stroke.Thickness = 2
        stroke.Parent = menuBtn

        Ext.makeTappable(menuBtn, toggleWindow)
    end)
    if not okMenu then warn("[ItemSystem] สร้างปุ่มเมนูลอยไม่สำเร็จ") end

    local Sec = newSection(Tabs.General, "ปุ่มเปิด/ปิดเมนู")
    local Tg = Sec:AddToggle("MenuButton", {
        Title = "แสดงปุ่มเมนูลอยบนหน้าจอ",
        Description = "กดเพื่อซ่อน/แสดงหน้าต่างสคริปต์ ใช้บนมือถือได้ (ไม่ต้องใช้ RightControl) ลากย้ายตำแหน่งได้",
        Default = (Ext.platform == "mobile"),
    })
    Tg:OnChanged(function()
        if menuGui then menuGui.Enabled = Options.MenuButton.Value end
    end)
    if menuGui then menuGui.Enabled = Options.MenuButton.Value end

    function Ext.menuCleanup()
        if menuGui then
            menuGui:Destroy()
            menuGui = nil
        end
    end
end

-- ========== จำค่าอัตโนมัติ (บันทึกทุกครั้งที่เปลี่ยน / โหลดให้เองตอนเข้าเกมใหม่) ==========
do
    local AUTO_NAME = "autosave"

    local function configPath()
        return tostring(SaveManager and SaveManager.Folder or "ItemManagementSystem/config")
            .. "/settings/" .. AUTO_NAME .. ".json"
    end

    -- โหลดค่าที่เคยจำไว้ (ถ้าผู้ใช้ไม่ได้ตั้ง autoload config ของ SaveManager เอง)
    local loaded = false
    if SaveManager then
        local hasAutoload = false
        pcall(function()
            hasAutoload = isfile(tostring(SaveManager.Folder) .. "/settings/autoload.txt")
        end)
        if not hasAutoload then
            local ok, res = pcall(function() return SaveManager:Load(AUTO_NAME) end)
            loaded = ok and res == true
        end
    end
    if loaded then notify("จำค่าอัตโนมัติ", "โหลดค่าที่เคยตั้งไว้แล้ว", 4) end

    local Sec = newSection(Tabs.Settings, "จำค่าอัตโนมัติ")
    Sec:AddToggle("AutoSave", {
        Title = "จำค่าที่ตั้งไว้อัตโนมัติ",
        Description = "บันทึกทุกครั้งที่เปลี่ยนค่า เข้าเกม/แมพใหม่แล้วจะโหลดค่าเดิมให้เอง (เก็บในไฟล์ของเครื่องนี้ รวมถึง Webhook URL อย่าแชร์ไฟล์)",
        Default = true,
    })

    Sec:AddButton({
        Title = "บันทึกตอนนี้",
        Callback = function()
            local ok, res, err = pcall(function() return SaveManager:Save(AUTO_NAME) end)
            if ok and res then
                notify("จำค่าอัตโนมัติ", "บันทึกค่าแล้ว", 3)
            else
                notify("จำค่าอัตโนมัติ", "บันทึกไม่สำเร็จ: " .. tostring(err or res), 5)
            end
        end,
    })

    Sec:AddButton({
        Title = "ล้างค่าที่จำไว้",
        Description = "ลบไฟล์ที่จำไว้ ครั้งหน้าจะเริ่มจากค่าเริ่มต้น",
        Callback = function()
            pcall(function()
                if delfile and isfile(configPath()) then delfile(configPath()) end
            end)
            notify("จำค่าอัตโนมัติ", "ลบค่าที่จำไว้แล้ว", 4)
        end,
    })

    -- สรุปค่าทั้งหมดเป็นข้อความ ใช้เทียบว่ามีอะไรเปลี่ยนไหม
    local function signature()
        local parts = {}
        for name, opt in pairs(Options) do
            local v = opt.Value
            local text
            if typeof(v) == "table" then
                local keys = {}
                for k, on in pairs(v) do
                    if on then table.insert(keys, tostring(k)) end
                end
                table.sort(keys)
                text = table.concat(keys, ",")
            else
                text = tostring(v)
            end
            table.insert(parts, tostring(name) .. "=" .. text)
        end
        table.sort(parts)
        return table.concat(parts, "|")
    end

    task.spawn(function()
        task.wait(4) -- ให้ค่าที่โหลดเสร็จก่อน จะได้ไม่บันทึกทับตอนเริ่ม
        local last = signature()
        while not Fluent.Unloaded do
            task.wait(2)
            if SaveManager and Options.AutoSave and Options.AutoSave.Value then
                local sig = signature()
                if sig ~= last then
                    last = sig
                    pcall(function() SaveManager:Save(AUTO_NAME) end)
                end
            end
        end
    end)
end

-- ========== อ่านค่าจาก getgenv().Configs (รูปแบบเดียวกับคอนฟิกฟาร์ม) ==========
-- ใช้เฉพาะคีย์ที่สคริปต์นี้รองรับ: Fishing.Filter.MinRarity, Fishing.Webhook, Settings.*, Underground.Depth
-- ค่าจาก Configs ถูกใช้ทับค่าที่จำไว้ทุกครั้งที่รัน (คีย์อื่นในคอนฟิกถูกข้าม)
do
    local okC, C = pcall(function() return getgenv().Configs end)
    if okC and type(C) == "table" then
        local function get(...)
            local node = C
            for _, k in ipairs({ ... }) do
                if type(node) ~= "table" then return nil end
                node = node[k]
            end
            return node
        end
        local function set(name, value)
            pcall(function() Options[name]:SetValue(value) end)
        end

        -- ปลา: Webhook ตามระดับ
        local rarities = get("Fishing", "Webhook", "Rarities")
        if type(rarities) == "table" then
            local lowered = {}
            for name, on in pairs(rarities) do
                if on then lowered[string.lower(tostring(name))] = true end
            end
            Ext.Fish.WebhookRarities = lowered
        end
        local url = get("Fishing", "Webhook", "Url")
        if type(url) == "string" and url:gsub("%s", "") ~= "" then
            set("DiscordWebhookURL", url)
        end
        if get("Fishing", "Webhook", "Enabled") == true then
            set("FishLog", true)
            set("FishWebhook", true)
        end

        -- FPS / กราฟิก
        if get("Settings", "FramerateCap") == true then
            local limit = tonumber(get("Settings", "FramerateLimit"))
            if limit then set("FpsCap", math.clamp(limit, 15, 240)) end
            set("FpsLock", true)
        end
        if get("Settings", "Performance") == true then set("PerfMode", "Low") end

        -- มุดดิน: ความลึก
        local depth = tonumber(get("Underground", "Depth"))
        if depth then set("BurrowDepth", math.clamp(depth, 3, 100)) end

        notify("Configs", "โหลดค่าจาก Configs แล้ว", 4)
    end
end

-- แปลหน้าต่างตามภาษาที่เลือกไว้ (ทำหลังสร้าง UI และโหลดค่าเสร็จ)
pcall(Ext.apply)
task.delay(2, function() pcall(Ext.apply) end)

--// ============================== ลูปหลัก ==============================

-- ลูป Auto-Collect
task.spawn(function()
    while not Fluent.Unloaded do
        if State.CollectEnabled then
            local ok, err = pcall(scanOnce)
            if not ok then warn("[ItemSystem] scan error:", err) end
        end
        task.wait(CONFIG.SCAN_INTERVAL)
    end
end)

-- ลูป Player ESP
task.spawn(function()
    while not Fluent.Unloaded do
        if State.PlayerESPEnabled then
            local ok, err = pcall(updatePlayerESP)
            if not ok then warn("[ItemSystem] PlayerESP error:", err) end
        end
        task.wait(0.45)
    end
end)

-- ลูป ESP
task.spawn(function()
    while not Fluent.Unloaded do
        if State.ESPEnabled then
            local ok, err = pcall(updateESP)
            if not ok then warn("[ItemSystem] ESP error:", err) end
        end
        task.wait(CONFIG.ESP_INTERVAL)
    end
end)

-- ตัวนับ FPS
do
    Stats.FpsFrames = 0
    Stats.FpsConn = RunService.RenderStepped:Connect(function()
        Stats.FpsFrames = Stats.FpsFrames + 1
    end)
    task.spawn(function()
        while not Fluent.Unloaded do
            task.wait(1)
            setParagraph(FpsPara, tostring(Stats.FpsFrames) .. " FPS")
            Stats.FpsFrames = 0
        end
        if Stats.FpsConn then
            pcall(function() Stats.FpsConn:Disconnect() end)
            Stats.FpsConn = nil
        end
    end)
end

-- เมื่อปิดสคริปต์ (Unload Fluent) ให้เก็บกวาดทุกอย่างคืนสภาพเดิม
task.spawn(function()
    while not Fluent.Unloaded do
        task.wait(0.5)
    end
    State.CollectEnabled = false
    State.ESPEnabled = false
    State.PlayerESPEnabled = false
    State.NoRecoil = false
    State.InfStamina = false
    pcall(restoreGuns)
    -- [FIXED] ตัด connection ที่เฝ้าปืน/ตัวละครเมื่อ Unload ไม่ให้สะสมหากรันไฟล์ซ้ำ
    for container, conn in pairs(GunContainerConnections) do
        pcall(function() conn:Disconnect() end)
        GunContainerConnections[container] = nil
    end
    for _, conn in ipairs(CombatLifecycleConnections) do
        if conn then pcall(function() conn:Disconnect() end) end
    end
    table.clear(CombatLifecycleConnections)
    pcall(function() RunService:UnbindFromRenderStep("IMS_Aimbot_CameraLock_v16") end)
    if AimGui then pcall(function() AimGui:Destroy() end) end
    pcall(Ext.burrowCleanup)
    pcall(Ext.menuCleanup)
    pcall(clearAllPlayerESP)
    pcall(stopVacuum)
    pcall(clearAllESP)
    pcall(setPerfLevel, 0)
    pcall(setFpsLock, false, FpsLock.Cap)
    if Stats.FpsConn then pcall(function() Stats.FpsConn:Disconnect() end); Stats.FpsConn = nil end
    table.clear(Stats.PlayerItemCache)
    table.clear(Stats.ZoneCache)
    table.clear(Stats.AnchorCache)
    table.clear(Stats.ItemInfoCache)
    for player in pairs(DeathMarkers) do pcall(removeDeathMarker, player) end
    for player, conns in pairs(PlayerLifecycle) do
        for _, c in pairs(conns) do pcall(function() c:Disconnect() end) end
        PlayerLifecycle[player] = nil
    end
    if LocalLifecycle then pcall(function() LocalLifecycle:Disconnect() end) end
    if LocalCharacterRemoving then
        pcall(function() LocalCharacterRemoving:Disconnect() end)
        LocalCharacterRemoving = nil
    end
    -- [FIXED] Disconnect global player event listeners so re-running the script does not stack handlers.
    for _, conn in ipairs({ItemESPPlayerRemovingConnection, PlayerLifecycleAddedConnection, PlayerLifecycleRemovingConnection}) do
        if conn then pcall(function() conn:Disconnect() end) end
    end
end)


--// ===================== Add-on v16: ขนาดปุ่ม / ควบคุมรถ / ตรวจระบบตอนรัน =====================
--[[
  IMS Add-on v16  —  วาง "ต่อท้ายสุดของไฟล์สคริปต์หลัก" (ต่อจากบรรทัดสุดท้ายเดิม ไม่ต้องลบอะไรก่อน)
  ใช้ตัวแปรของสคริปต์หลักได้โดยตรง (Tabs, Options, Fluent, notify, newSection, getRoot, getHumanoid, setParagraph, HttpService, RunService ...)

  เพิ่มให้:
    1) สไลเดอร์ปรับขนาดปุ่ม IMS และปุ่ม Snap (จำค่าไว้ในไฟล์ของเครื่องนี้ ค่าเริ่มต้นเล็กลงกว่าเดิม)
    2) ระบบช่วยขับรถ: เลี้ยวไวขึ้น/ตอบสนองทันที (ปรับความแรงได้ จำค่าไว้)
    3) ตรวจสอบระบบเกมอัตโนมัติตอนรันสคริปต์ (ไม่ต้องกดปุ่มเอง)
]]
Ext.__a16 = (function()
    local STORE_FILE = "ItemManagementSystem/addon.json"
    local store = {}
    pcall(function()
        if isfile and isfile(STORE_FILE) then
            local data = HttpService:JSONDecode(readfile(STORE_FILE))
            if type(data) == "table" then store = data end
        end
    end)
    local function saveStore()
        store.Version = 3
        pcall(function()
            if makefolder and isfolder and not isfolder("ItemManagementSystem") then
                makefolder("ItemManagementSystem")
            end
            if writefile then writefile(STORE_FILE, HttpService:JSONEncode(store)) end
        end)
    end

    ------------------------------------------------------------------
    -- 1) ขนาดปุ่ม IMS / Snap
    ------------------------------------------------------------------
    local function guiRoots()
        local roots = {}
        local function add(x)
            if typeof(x) == "Instance" then table.insert(roots, x) end
        end
        add(LocalPlayer:FindFirstChild("PlayerGui"))
        pcall(function() add(game:GetService("CoreGui")) end)
        pcall(function() if gethui then add(gethui()) end end)
        return roots
    end

    local function findSnapButton()
        for _, root in ipairs(guiRoots()) do
            local g = root:FindFirstChild("IMS_Snap")
            local b = g and g:FindFirstChild("SnapButton")
            if b then return b end
        end
        return nil
    end

    -- ปุ่ม IMS (เปิด/ปิดหน้าต่าง): หา ScreenGui ที่ชื่อมีคำว่า IMS (ยกเว้น Snap/ตัวเลือกอุปกรณ์)
    local function findMenuButtons()
        local list = {}
        for _, root in ipairs(guiRoots()) do
            for _, g in ipairs(root:GetChildren()) do
                if g:IsA("ScreenGui") and string.find(g.Name, "IMS", 1, true)
                    and g.Name ~= "IMS_Snap" and g.Name ~= "IMS_DevicePicker" then
                    for _, d in ipairs(g:GetDescendants()) do
                        if d:IsA("GuiButton") then table.insert(list, d) end
                    end
                end
            end
        end
        if #list == 0 then
            -- สำรอง: หาปุ่มที่เขียนว่า IMS หรือชื่อมี IMS
            for _, root in ipairs(guiRoots()) do
                for _, d in ipairs(root:GetDescendants()) do
                    if d:IsA("GuiButton") and d.Name ~= "SnapButton"
                        and (string.find(d.Name, "IMS", 1, true) or (d:IsA("TextButton") and d.Text == "IMS")) then
                        table.insert(list, d)
                    end
                end
            end
        end
        return list
    end

    local sizes = {
        IMS = tonumber(store.IMSSize) or 52,
        Snap = tonumber(store.SnapSize) or 64,
    }

    local function applySizes()
        local snap = findSnapButton()
        if snap then
            snap.Size = UDim2.fromOffset(sizes.Snap, sizes.Snap)
            pcall(function() snap.TextSize = math.clamp(math.floor(sizes.Snap * 0.2), 9, 18) end)
        end
        for _, b in ipairs(findMenuButtons()) do
            b.Size = UDim2.fromOffset(sizes.IMS, sizes.IMS)
        end
    end

    local SecBtn = newSection(Tabs.General, "ปุ่มบนมือถือ (Mobile buttons)")

    local ImsSlider = SecBtn:AddSlider("IMSSize", {
        Title = "ขนาดปุ่ม IMS (IMS button size)",
        Description = "ปุ่มกลมที่แตะเพื่อเปิด/ปิดเมนู หน่วยพิกเซล",
        Default = sizes.IMS,
        Min = 30,
        Max = 100,
        Rounding = 0,
        Callback = function() end,
    })
    ImsSlider:OnChanged(function(v)
        sizes.IMS = v
        store.IMSSize = v
        saveStore()
        applySizes()
    end)

    local SnapSlider = SecBtn:AddSlider("SnapSize", {
        Title = "ขนาดปุ่ม Snap (Snap button size)",
        Description = "ปุ่มมุดดินบนมือถือ หน่วยพิกเซล",
        Default = sizes.Snap,
        Min = 36,
        Max = 120,
        Rounding = 0,
        Callback = function() end,
    })
    SnapSlider:OnChanged(function(v)
        sizes.Snap = v
        store.SnapSize = v
        saveStore()
        applySizes()
    end)

    -- ปุ่มถูกสร้างทีหลัง/ถูกสร้างใหม่ได้ จึงปรับซ้ำเป็นระยะ (เบามาก)
    task.spawn(function()
        for _ = 1, 8 do
            if Fluent.Unloaded then return end
            pcall(applySizes)
            task.wait(0.75)
        end
        while not Fluent.Unloaded do
            pcall(applySizes)
            task.wait(3)
        end
    end)

    ------------------------------------------------------------------
    -- 2) ระบบช่วยขับรถ (VehicleSeat)
    ------------------------------------------------------------------
    -- รถในเกมใช้ VehicleSeat มาตรฐานของ Roblox การเลี้ยวมาจาก SteerFloat และ TurnSpeed ของที่นั่ง
    -- ระบบนี้: (ก) ขยายค่าเลี้ยวที่เรากด ให้รถตอบสนองไวขึ้น (ข) เพิ่ม TurnSpeed ของที่นั่ง
    -- แก้เฉพาะฝั่งเครื่องเรา ไม่ได้เพิ่มความเร็วรถ และคืนค่าเดิมเมื่อปิด/ลงจากรถ
    local V = {
        Enabled = store.VehEnabled == true,
        Gain = tonumber(store.VehGain) or 1.6,
        Turn = tonumber(store.VehTurn) or 1.5,
        Orig = setmetatable({}, { __mode = "k" }),
        Seat = nil,
    }

    local function restoreSeat()
        for seat, orig in pairs(V.Orig) do
            pcall(function() seat.TurnSpeed = orig end)
            V.Orig[seat] = nil
        end
    end

    local function currentSeat()
        local hum = getHumanoid()
        local seat = hum and hum.SeatPart
        if seat and seat:IsA("VehicleSeat") then return seat end
        return nil
    end

    -- [FIXED] เก็บ connection ไว้และ disconnect ตัวเองหลัง UI Unload; เดิม callback ค้างทุก Heartbeat
    V.HeartbeatConnection = RunService.Heartbeat:Connect(function()
        if Fluent.Unloaded then
            restoreSeat()
            if V.HeartbeatConnection then
                pcall(function() V.HeartbeatConnection:Disconnect() end)
                V.HeartbeatConnection = nil
            end
            return
        end
        local seat = currentSeat()
        if V.Seat and V.Seat ~= seat then restoreSeat() end -- ลงจากรถ/เปลี่ยนรถ -> คืนค่าเดิม
        V.Seat = seat
        if not (V.Enabled and seat) then return end
        pcall(function()
            if V.Orig[seat] == nil then V.Orig[seat] = seat.TurnSpeed end
            seat.TurnSpeed = V.Orig[seat] * V.Turn
            local steer = seat.SteerFloat
            if steer ~= 0 then
                seat.SteerFloat = math.clamp(steer * V.Gain, -1, 1)
            end
        end)
    end)

    local SecCar = newSection(Tabs.Movement, "ควบคุมรถ (Car assist)")

    local CarPara = Tabs.Movement:AddParagraph({
        Title = "สถานะรถ (Car status)",
        Content = "ยังไม่ได้ขับรถ (not driving)",
    })

    local CarToggle = SecCar:AddToggle("CarAssist", {
        Title = "เปิดระบบช่วยขับรถ (Enable car assist)",
        Description = "เลี้ยวไวขึ้น ตอบสนองทันที ไม่ต้องกดค้างรอให้รถค่อย ๆ หัน",
        Default = V.Enabled,
    })
    CarToggle:OnChanged(function()
        V.Enabled = Options.CarAssist.Value
        store.VehEnabled = V.Enabled
        saveStore()
        if not V.Enabled then restoreSeat() end
    end)

    local GainSlider = SecCar:AddSlider("CarSteerGain", {
        Title = "ความไวเลี้ยว (Steering gain)",
        Description = "1.0 = ปกติ, สูงขึ้น = เลี้ยวตอบสนองไวขึ้น",
        Default = V.Gain,
        Min = 1,
        Max = 3,
        Rounding = 1,
        Callback = function() end,
    })
    GainSlider:OnChanged(function(v)
        V.Gain = v
        store.VehGain = v
        saveStore()
    end)

    local TurnSlider = SecCar:AddSlider("CarTurnSpeed", {
        Title = "ความเร็วหมุนพวงมาลัย (Turn speed)",
        Description = "คูณค่า TurnSpeed ของที่นั่งรถ 1.0 = ปกติ",
        Default = V.Turn,
        Min = 1,
        Max = 3,
        Rounding = 1,
        Callback = function() end,
    })
    TurnSlider:OnChanged(function(v)
        V.Turn = v
        store.VehTurn = v
        saveStore()
    end)

    task.spawn(function()
        while not Fluent.Unloaded do
            local seat = currentSeat()
            local text
            if seat then
                text = string.format("กำลังขับรถ (driving): %s | ระบบช่วย: %s",
                    tostring(seat.Parent and seat.Parent.Name or seat.Name), V.Enabled and "เปิด (on)" or "ปิด (off)")
            else
                text = "ยังไม่ได้ขับรถ (not driving)"
            end
            pcall(setParagraph, CarPara, text)
            task.wait(1)
        end
    end)

    ------------------------------------------------------------------
    -- 3) ตรวจสอบระบบเกมอัตโนมัติตอนรัน (ไม่ต้องกดปุ่มตรวจจับเอง)
    ------------------------------------------------------------------
    task.spawn(function()
        task.wait(2.5) -- รอให้เกม/โมดูลโหลดก่อน
        if Fluent.Unloaded then return end
        local ok = pcall(function() refreshDetect(true) end)
        if not ok then warn("[IMS Addon] refreshDetect ไม่พร้อมใช้งาน") end
    end)
end)()


--// ===================== Add-on v17: Chams / เพื่อน / เส้นกระสุน =====================
--[[
  IMS Add-on v17 — Chams + ตัวที่เล็งเป็นสีแดง + เพื่อนสีเขียว + เส้นกระสุน
  วาง "ต่อท้ายสุดของไฟล์สคริปต์หลัก" (ต่อจาก Add-on v16 ถ้าใส่ไว้แล้ว) ไม่ต้องลบอะไร
  ใช้ตัวแปรของสคริปต์หลัก: Tabs, Options, Fluent, notify, newSection, getRoot, Players, Workspace, RunService,
  UserInputService, IS_MOBILE, LocalPlayer, HttpService

  ทำอะไร:
    - ทุกคนในเซิร์ฟเห็นเป็นตัวสีขาวทะลุกำแพง (Highlight)
    - คนที่กึ่งกลางจอ/เมาส์ชี้อยู่ กลายเป็นสีแดง (แค่ไฮไลต์ ไม่ได้ล็อกหรือช่วยเล็งอะไร)
    - เพื่อน (เพื่อน Roblox + รายชื่อที่ใส่เอง) เป็นสีเขียวเสมอ ไม่เป็นสีแดง
    - ตอนยิงมีเส้นสีแดงจากปลายปืนไปจุดที่เล็ง
  หมายเหตุ: Roblox แสดง Highlight พร้อมกันได้ราว 31 อัน คนที่เกินจะไม่มีสี
]]
Ext.__a17 = (function()
    local STORE_FILE = "ItemManagementSystem/chams.json"
    local store = {}
    pcall(function()
        if isfile and isfile(STORE_FILE) then
            local data = HttpService:JSONDecode(readfile(STORE_FILE))
            if type(data) == "table" then store = data end
        end
    end)
    local function saveStore()
        pcall(function()
            if makefolder and isfolder and not isfolder("ItemManagementSystem") then
                makefolder("ItemManagementSystem")
            end
            if writefile then writefile(STORE_FILE, HttpService:JSONEncode(store)) end
        end)
    end

    local WHITE = Color3.fromRGB(255, 255, 255)
    local RED   = Color3.fromRGB(255, 40, 40)
    local GREEN = Color3.fromRGB(60, 255, 90)

    local C = {
        Enabled = store.ChamsOn == true,
        Fill = tonumber(store.Fill) or 0.55,
        Range = tonumber(store.Range) or 2000,
        ShowTarget = store.ShowTarget ~= false,
        UseFriends = store.UseFriends ~= false,
        FriendNames = {},
        Tracer = store.TracerOn == true,
        TracerTime = tonumber(store.TracerTime) or 0.15,
        Target = nil,
    }

    local function parseNames(text)
        local set = {}
        for name in string.gmatch(tostring(text or ""), "[^,%s]+") do
            set[string.lower(name)] = true
        end
        return set
    end
    C.FriendNames = parseNames(store.FriendNames)

    ------------------------------------------------------------------
    -- เพื่อน
    ------------------------------------------------------------------
    local friendCache = {}   -- [userId] = true/false
    local friendAsked = {}

    local function requestFriend(player)
        if friendAsked[player.UserId] then return end
        friendAsked[player.UserId] = true
        task.spawn(function()
            local ok, res = pcall(function() return LocalPlayer:IsFriendsWith(player.UserId) end)
            friendCache[player.UserId] = ok and res == true
        end)
    end

    local function isFriend(player)
        if C.FriendNames[string.lower(player.Name)] or C.FriendNames[string.lower(player.DisplayName)] then
            return true
        end
        if C.UseFriends then
            requestFriend(player)
            return friendCache[player.UserId] == true
        end
        return false
    end

    ------------------------------------------------------------------
    -- ยิงเรย์จากกึ่งกลางจอ/เมาส์ เพื่อดูว่าชี้อยู่ที่ใคร และจุดที่เล็งอยู่ตรงไหน
    ------------------------------------------------------------------
    local function aimRay()
        local cam = Workspace.CurrentCamera
        if not cam then return nil end
        local loc
        if not IS_MOBILE and UserInputService.MouseBehavior == Enum.MouseBehavior.Default then
            loc = UserInputService:GetMouseLocation()
        else
            loc = cam.ViewportSize / 2
        end
        local ray = cam:ViewportPointToRay(loc.X, loc.Y)
        return ray.Origin, ray.Direction * 1500
    end

    local function aimHit()
        local origin, dir = aimRay()
        if not origin then return nil, nil end
        local params = RaycastParams.new()
        params.FilterType = Enum.RaycastFilterType.Exclude
        params.FilterDescendantsInstances = { LocalPlayer.Character }
        local hit = Workspace:Raycast(origin, dir, params)
        local point = hit and hit.Position or (origin + dir)
        local who
        if hit then
            local node = hit.Instance
            while node and node ~= Workspace do
                local p = Players:GetPlayerFromCharacter(node)
                if p then who = p break end
                node = node.Parent
            end
        end
        return point, who
    end

    ------------------------------------------------------------------
    -- Chams (Highlight)
    ------------------------------------------------------------------
    local holder
    do
        local parent
        pcall(function() if gethui then parent = gethui() end end)
        if not parent then pcall(function() parent = game:GetService("CoreGui") end) end
        holder = Instance.new("Folder")
        holder.Name = "IMS_Chams"
        local ok = pcall(function() holder.Parent = parent end)
        if not ok or not holder.Parent then holder.Parent = Workspace.CurrentCamera or Workspace end
    end

    local highlights = {}  -- [player] = Highlight

    local function removeHighlight(player)
        local h = highlights[player]
        highlights[player] = nil
        if h then pcall(function() h:Destroy() end) end
    end

    local function clearAll()
        local list = {}
        for player in pairs(highlights) do table.insert(list, player) end
        for _, player in ipairs(list) do removeHighlight(player) end
    end

    local function colorFor(player)
        if isFriend(player) then return GREEN end
        if C.ShowTarget and C.Target == player then return RED end
        return WHITE
    end

    local function updateChams()
        local myRoot = getRoot()
        local alive = {}
        if myRoot then
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= LocalPlayer then
                    local char = player.Character
                    local root = char and char:FindFirstChild("HumanoidRootPart")
                    local hum = char and char:FindFirstChildOfClass("Humanoid")
                    if root and hum and hum.Health > 0
                        and (root.Position - myRoot.Position).Magnitude <= C.Range then
                        alive[player] = true
                        local h = highlights[player]
                        if h and h.Adornee ~= char then
                            removeHighlight(player) -- เกิดใหม่: สร้างใหม่
                            h = nil
                        end
                        if not h then
                            h = Instance.new("Highlight")
                            h.Name = "IMS_Cham"
                            h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                            h.Adornee = char
                            h.Parent = holder
                            highlights[player] = h
                        end
                        local color = colorFor(player)
                        h.FillColor = color
                        h.OutlineColor = color
                        h.FillTransparency = C.Fill
                        h.OutlineTransparency = 0
                    end
                end
            end
        end
        local stale = {}
        for player in pairs(highlights) do
            if not alive[player] then table.insert(stale, player) end
        end
        for _, player in ipairs(stale) do removeHighlight(player) end
    end

    local ChamsPlayerRemovingConnection = Players.PlayerRemoving:Connect(function(player)
        removeHighlight(player)
        friendCache[player.UserId] = nil
        friendAsked[player.UserId] = nil
    end)

    task.spawn(function()
        while not Fluent.Unloaded do
            if C.Enabled then
                local _, who = aimHit()
                C.Target = who
                pcall(updateChams)
            else
                C.Target = nil
                if next(highlights) then clearAll() end
            end
            task.wait(0.06)
        end
        clearAll()
        pcall(function() holder:Destroy() end)
        -- [FIXED] PlayerRemoving ไม่ได้ผูกกับ Instance ที่ถูก Destroy; ตัด connection เอง
        if ChamsPlayerRemovingConnection then
            pcall(function() ChamsPlayerRemovingConnection:Disconnect() end)
        end
    end)

    ------------------------------------------------------------------
    -- เส้นกระสุน (Tracer) สีแดง
    ------------------------------------------------------------------
    local lastTracer = 0

    local function drawTracer()
        if not C.Tracer then return end
        local now = os.clock()
        if now - lastTracer < 0.04 then return end
        lastTracer = now

        local char = LocalPlayer.Character
        local tool = char and char:FindFirstChildOfClass("Tool")
        if not tool then return end
        local handle = tool:FindFirstChild("Handle") or tool:FindFirstChildWhichIsA("BasePart", true)
        local root = getRoot()
        local from = handle and handle.Position or (root and root.Position)
        local to = aimHit()
        if not from or not to then return end

        local dist = (to - from).Magnitude
        if dist < 1 then return end

        local beam = Instance.new("Part")
        beam.Anchored = true
        beam.CanCollide = false
        beam.CanQuery = false
        beam.CanTouch = false
        beam.CastShadow = false
        beam.Material = Enum.Material.Neon
        beam.Color = RED
        beam.Size = Vector3.new(0.07, 0.07, dist)
        beam.CFrame = CFrame.lookAt(from, to) * CFrame.new(0, 0, -dist / 2)
        beam.Parent = Workspace.CurrentCamera or Workspace
        task.delay(C.TracerTime, function() pcall(function() beam:Destroy() end) end)
    end

    -- [FIXED] De-duplicate tool hooks and retain global connections for Unload cleanup.
    local tracerToolConnections = {}
    local tracerCharacterConnections = {}
    local hookedTracerTools = setmetatable({}, {__mode = "k"})
    local function hookTool(tool)
        if tool:IsA("Tool") and not hookedTracerTools[tool] then
            hookedTracerTools[tool] = true
            table.insert(tracerToolConnections, tool.Activated:Connect(function() pcall(drawTracer) end))
        end
    end

    local function hookCharacter(char)
        for _, t in ipairs(char:GetChildren()) do hookTool(t) end
        table.insert(tracerCharacterConnections, char.ChildAdded:Connect(hookTool))
    end
    if LocalPlayer.Character then hookCharacter(LocalPlayer.Character) end
    local tracerCharacterAddedConnection = LocalPlayer.CharacterAdded:Connect(hookCharacter)

    -- เผื่อเกมยิงด้วยระบบรับปุ่มของตัวเอง (ไม่ผ่าน Tool.Activated): ดูการกดเมาส์ซ้าย ตอนถือของอยู่
    local holding = false
    local tracerInputBeganConnection = UserInputService.InputBegan:Connect(function(input, processed)
        if Fluent.Unloaded or processed or not C.Tracer then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            holding = true
            task.spawn(function()
                while holding and C.Tracer and not Fluent.Unloaded do
                    pcall(drawTracer)
                    task.wait(0.1)
                end
            end)
        end
    end)
    local tracerInputEndedConnection = UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then holding = false end
    end)
    task.spawn(function()
        while not Fluent.Unloaded do task.wait(0.5) end
        C.Tracer = false
        holding = false
        for _, conn in ipairs(tracerToolConnections) do pcall(function() conn:Disconnect() end) end
        for _, conn in ipairs(tracerCharacterConnections) do pcall(function() conn:Disconnect() end) end
        for _, conn in ipairs({tracerCharacterAddedConnection, tracerInputBeganConnection, tracerInputEndedConnection}) do
            if conn then pcall(function() conn:Disconnect() end) end
        end
        table.clear(tracerToolConnections)
        table.clear(tracerCharacterConnections)
    end)

    ------------------------------------------------------------------
    -- UI (แท็บ Player ESP)
    ------------------------------------------------------------------
    local Sec = newSection(Tabs.PlayerESP, "Chams (ตัวผู้เล่น)")

    local On = Sec:AddToggle("ChamsOn", {
        Title = "เปิด Chams (Enable chams)",
        Description = "ทุกคนในเซิร์ฟเป็นตัวสีขาวทะลุกำแพง",
        Default = C.Enabled,
    })
    On:OnChanged(function()
        C.Enabled = Options.ChamsOn.Value
        store.ChamsOn = C.Enabled
        saveStore()
    end)

    local Fill = Sec:AddSlider("ChamsFill", {
        Title = "ความโปร่งใสของสี (Fill transparency)",
        Description = "0 = ทึบ, 1 = เห็นแค่เส้นขอบ",
        Default = C.Fill,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function() end,
    })
    Fill:OnChanged(function(v)
        C.Fill = v
        store.Fill = v
        saveStore()
    end)

    local Range = Sec:AddSlider("ChamsRange", {
        Title = "ระยะแสดง Chams (Range)",
        Description = "หน่วย studs",
        Default = C.Range,
        Min = 100,
        Max = 5000,
        Rounding = 0,
        Callback = function() end,
    })
    Range:OnChanged(function(v)
        C.Range = v
        store.Range = v
        saveStore()
    end)

    local Tgt = Sec:AddToggle("ChamsTarget", {
        Title = "คนที่เล็งอยู่เป็นสีแดง (Red = who you aim at)",
        Description = "ไฮไลต์คนที่กึ่งกลางจอ/เมาส์ชี้อยู่เท่านั้น ไม่ได้ล็อกหรือช่วยเล็ง",
        Default = C.ShowTarget,
    })
    Tgt:OnChanged(function()
        C.ShowTarget = Options.ChamsTarget.Value
        store.ShowTarget = C.ShowTarget
        saveStore()
    end)

    local SecF = newSection(Tabs.PlayerESP, "เพื่อน (Friends = เขียว)")

    local Fr = SecF:AddToggle("ChamsFriends", {
        Title = "เพื่อน Roblox เป็นสีเขียว (Roblox friends = green)",
        Default = C.UseFriends,
    })
    Fr:OnChanged(function()
        C.UseFriends = Options.ChamsFriends.Value
        store.UseFriends = C.UseFriends
        saveStore()
    end)

    local Names = SecF:AddInput("ChamsFriendNames", {
        Title = "ชื่อเพื่อนเพิ่มเอง (Extra friend names)",
        Description = "ใส่ชื่อหรือ DisplayName คั่นด้วยเครื่องหมายจุลภาค เช่น Alex, Bob",
        Default = tostring(store.FriendNames or ""),
        Placeholder = "Alex, Bob",
        Numeric = false,
        Finished = false,
    })
    Names:OnChanged(function(v)
        C.FriendNames = parseNames(v)
        store.FriendNames = tostring(v or "")
        saveStore()
    end)

    local SecT = newSection(Tabs.PlayerESP, "เส้นกระสุน (Tracer)")

    local Tr = SecT:AddToggle("TracerOn", {
        Title = "เส้นสีแดงตอนยิง (Red shot line)",
        Description = "แสดงเส้นจากปลายปืนไปจุดที่เล็ง ตอนกดยิง",
        Default = C.Tracer,
    })
    Tr:OnChanged(function()
        C.Tracer = Options.TracerOn.Value
        store.TracerOn = C.Tracer
        saveStore()
    end)

    local Tt = SecT:AddSlider("TracerTime", {
        Title = "ระยะเวลาที่เส้นค้าง (วินาที)",
        Default = C.TracerTime,
        Min = 0.05,
        Max = 0.6,
        Rounding = 2,
        Callback = function() end,
    })
    Tt:OnChanged(function(v)
        C.TracerTime = v
        store.TracerTime = v
        saveStore()
    end)
end)()

