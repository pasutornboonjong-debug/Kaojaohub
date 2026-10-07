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

local LocalPlayer = Players.LocalPlayer

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
        { "ภาษา (Language)", "Language" }
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

Players.PlayerRemoving:Connect(function(player)
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
    if PlayerLifecycle[player] then
        for _, c in pairs(PlayerLifecycle[player]) do pcall(function() c:Disconnect() end) end
    end
    PlayerLifecycle[player] = {}

    local function bindCharacter(char)
        if not char then return end
        Stats.PlayerItemCache[player] = nil
        local humanoid = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 5)
        if not humanoid then return end
        table.insert(PlayerLifecycle[player], humanoid.Died:Connect(function()
            local root = char:FindFirstChild("HumanoidRootPart")
            if root then createDeathMarker(player, root.Position) end
            removePlayerESP(player)
        end))
    end

    if player.Character then bindCharacter(player.Character) end
    table.insert(PlayerLifecycle[player], player.CharacterAdded:Connect(function(char)
        removeDeathMarker(player)
        task.wait(0.15)
        bindCharacter(char)
    end))
end

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then bindPlayerLifecycle(player) end
end
Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then bindPlayerLifecycle(player) end
end)
Players.PlayerRemoving:Connect(function(player)
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
LocalPlayer.CharacterAdded:Connect(function(char)
    task.spawn(function()
        task.wait(0.1)
        bindLocalCharacter(char)
    end)
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
        ShowButton = UserInputService.TouchEnabled,
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

    -- ปุ่ม Snap บนหน้าจอ (สำหรับมือถือ) ลากย้ายตำแหน่งได้
    local okBtn = pcall(function()
        btnGui = Instance.new("ScreenGui")
        btnGui.Name = "IMS_Snap"
        btnGui.ResetOnSpawn = false
        btnGui.DisplayOrder = 20
        btnGui.Enabled = false
        btnGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

        btn = Instance.new("TextButton")
        btn.Size = UDim2.fromOffset(72, 72)
        btn.Position = UDim2.new(1, -110, 0.55, 0)
        btn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
        btn.BackgroundTransparency = 0.15
        btn.TextColor3 = Color3.new(1, 1, 1)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 16
        btn.Text = "Snap ▼"
        btn.Active = true
        pcall(function() btn.Draggable = true end)
        btn.Parent = btnGui

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1, 0)
        corner.Parent = btn
        local stroke = Instance.new("UIStroke")
        stroke.Color = Color3.fromRGB(120, 170, 255)
        stroke.Thickness = 2
        stroke.Parent = btn

        btn.Activated:Connect(function() B.toggle() end)
    end)
    if not okBtn then warn("[ItemSystem] สร้างปุ่ม Snap ไม่สำเร็จ") end

    UserInputService.InputBegan:Connect(function(input, processed)
        if processed or Fluent.Unloaded then return end
        if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == B.KeyCode then
            B.toggle()
        end
    end)
end

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
Window = Fluent:CreateWindow({
    Title = "Item Management System",
    SubTitle = "Auto-Collect + ESP",
    TabWidth = 160,
    Size = UDim2.fromOffset(580, 460),
    Acrylic = false, -- ปิดเบลอ (เบลออาจถูกตรวจจับได้และกินเครื่อง)
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.RightControl,
})

local Tabs = {
    Main     = Window:AddTab({ Title = "Main", Icon = "" }),
    PlayerESP = Window:AddTab({ Title = "Player ESP", Icon = "" }),
    Combat   = Window:AddTab({ Title = "Combat", Icon = "" }),
    Movement = Window:AddTab({ Title = "Movement", Icon = "" }),
    General  = Window:AddTab({ Title = "General", Icon = "" }),
    Settings = Window:AddTab({ Title = "Settings", Icon = "settings" }),
}

local Options = Fluent.Options

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
        pcall(scanGuns)
    else
        restoreGuns()
    end
end)

local StaminaToggle = SecCombat:AddToggle("InfStamina", {
    Title = "สเตมิน่าไม่จำกัด",
    Description = "ดันแถบสเตมิน่าให้เต็มตลอด วิ่ง/กระโดดได้เรื่อย ๆ",
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

task.spawn(function()
    task.wait(3)
    pcall(refreshDetect, false)
end)

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
        SaveManager:SetIgnoreIndexes({ "Language" })
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
    pcall(Ext.burrowCleanup)
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
end)
