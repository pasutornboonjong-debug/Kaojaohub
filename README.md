--[[
    Item Management System (Fluent UI) - รุ่น v15 (Mobile Fix, IMS รูปภาพ, Snap แตะครั้งเดียว)
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
local UserInputService = game:GetService("UserInputService")
local IS_MOBILE = UserInputService.TouchEnabled

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
        { "กดเพื่อซ่อน/แสดงหน้าต่างสคริปต์ ใช้บนมือถือได้ (ไม่ต้องใช้ RightControl) ลากย้ายตำแหน่งได้", "Tap to hide/show the script window. Works on mobile (no RightControl needed); drag to move it" }
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

--// ============================== Responsive UI สำหรับมือถือ ==============================
local MobileWindowScaleConn
do
    if IS_MOBILE then
        local function applyMobileWindowScale()
            local root = findFluentRoot()
            local camera = Workspace.CurrentCamera
            if not root or not camera then return end

            local vp = camera.ViewportSize
            local sx = (vp.X * 0.92) / 580
            local sy = (vp.Y * 0.84) / 460
            local scale = math.clamp(math.min(sx, sy), 0.62, 0.92)

            local ui = root:FindFirstChildOfClass("UIScale")
            if not ui then
                ui = Instance.new("UIScale")
                ui.Parent = root
            end
            ui.Scale = scale
        end

        pcall(applyMobileWindowScale)
        pcall(function()
            if Workspace.CurrentCamera then
                MobileWindowScaleConn = Workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(function()
                    pcall(applyMobileWindowScale)
                end)
            end
        end)
    end
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
    local origCollision = setmetatable({}, { __mode = "k" })
    local btnGui, btn

    local function refreshButton()
        if btnGui then btnGui.Enabled = IS_MOBILE and B.Enabled and B.ShowButton end
        if btn then btn.Text = B.Active and "SNAP\n▲ ขึ้น" or "SNAP\n▼ ลง" end
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

        -- คืนค่า CanCollide ของชิ้นส่วนตัวละครให้ครบ ไม่ใช่แค่ HumanoidRootPart
        for part, oldValue in pairs(origCollision) do
            pcall(function() part.CanCollide = oldValue end)
            origCollision[part] = nil
        end

        B.Active = false
        startChar = nil
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
                if part:IsA("BasePart") then
                    if origCollision[part] == nil then
                        origCollision[part] = part.CanCollide
                    end
                    part.CanCollide = false
                end
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
        -- Snap เป็นปุ่มสำหรับมือถือเท่านั้น บนคอมให้ใช้ปุ่ม Z ตามเดิม
        B.ShowButton = IS_MOBILE and (value and true or false) or false
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

    -- ปุ่ม Snap สำหรับมือถือ: ปุ่มใหญ่ แตะครั้งเดียว ทำงานทันที และอยู่เหนือ UI ของเกม
    local okBtn = pcall(function()
        btnGui = Instance.new("ScreenGui")
        btnGui.Name = "IMS_Snap"
        btnGui.ResetOnSpawn = false
        btnGui.IgnoreGuiInset = true
        btnGui.DisplayOrder = 10000
        btnGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
        btnGui.Enabled = false
        btnGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

        btn = Instance.new("TextButton")
        btn.Name = "SnapButton"
        btn.AnchorPoint = Vector2.new(1, 0.5)
        btn.Size = UDim2.fromOffset(88, 88)
        btn.Position = UDim2.new(1, -20, 0.55, 0)
        btn.BackgroundColor3 = Color3.fromRGB(70, 90, 180)
        btn.BackgroundTransparency = 0.04
        btn.TextColor3 = Color3.new(1, 1, 1)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 17
        btn.TextWrapped = true
        btn.Text = "SNAP\n▼ ลง"
        btn.AutoButtonColor = true
        btn.Active = true
        btn.Selectable = true
        btn.ZIndex = 10
        btn.Parent = btnGui

        local gradient = Instance.new("UIGradient")
        gradient.Rotation = 35
        gradient.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 220, 255)),
            ColorSequenceKeypoint.new(0.5, Color3.fromRGB(115, 80, 255)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 70, 170)),
        })
        gradient.Parent = btn

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(1, 0)
        corner.Parent = btn

        local stroke = Instance.new("UIStroke")
        stroke.Color = Color3.fromRGB(255, 255, 255)
        stroke.Transparency = 0.18
        stroke.Thickness = 2.5
        stroke.Parent = btn

        local padding = Instance.new("UIPadding")
        padding.PaddingTop = UDim.new(0, 8)
        padding.PaddingBottom = UDim.new(0, 8)
        padding.PaddingLeft = UDim.new(0, 6)
        padding.PaddingRight = UDim.new(0, 6)
        padding.Parent = btn

        btn.Activated:Connect(function()
            if Fluent.Unloaded then return end
            if B.Enabled then
                B.toggle()
            else
                notify("มุดดิน", "กรุณาเปิดระบบมุดดินก่อน", 3)
            end
        end)
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

notify("Item Management System", IS_MOBILE and "โหลดสำเร็จ แตะปุ่ม IMS ครั้งเดียวเพื่อเปิด/ปิด UI" or "โหลดสำเร็จ กด RightControl เพื่อซ่อน/แสดงหน้าต่าง", 6)

pcall(function()
    if SaveManager then SaveManager:LoadAutoloadConfig() end
end)



-- ========== รูปปุ่ม IMS สำหรับมือถือ ==========
-- ฝังรูปเป็น Base64 เพื่อให้ไฟล์สคริปต์ตัวเดียวสร้าง asset ให้ executor ได้เอง
local IMS_BUTTON_BASE64 = [[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]]

local function decodeIMSBase64(data)
    -- ลอง decoder ของ executor ก่อน เพราะเร็วกว่า
    local candidates = {
        function()
            local c = rawget(_G, "crypt")
            return c and c.base64 and c.base64.decode and c.base64.decode(data)
        end,
        function()
            local s = rawget(_G, "syn")
            return s and s.crypt and s.crypt.base64 and s.crypt.base64.decode and s.crypt.base64.decode(data)
        end,
        function()
            local b = rawget(_G, "base64")
            return b and b.decode and b.decode(data)
        end,
    }
    for _, fn in ipairs(candidates) do
        local ok, result = pcall(fn)
        if ok and type(result) == "string" and #result > 0 then
            return result
        end
    end

    -- fallback decoder แบบ Lua ล้วน
    local alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    data = tostring(data):gsub("[^%w%+/%=]", "")
    return (data:gsub(".", function(x)
        if x == "=" then return "" end
        local r, f = "", (alphabet:find(x, 1, true) or 1) - 1
        for i = 6, 1, -1 do
            r = r .. ((f % 2^i - f % 2^(i - 1) > 0) and "1" or "0")
        end
        return r
    end):gsub("%d%d%d?%d?%d?%d?%d?%d?", function(x)
        if #x ~= 8 then return "" end
        local c = 0
        for i = 1, 8 do c = c * 2 + (x:sub(i, i) == "1" and 1 or 0) end
        return string.char(c)
    end))
end

local function getIMSButtonAsset()
    if type(writefile) ~= "function" then return nil end
    local getter = (type(getcustomasset) == "function" and getcustomasset)
        or (type(getsynasset) == "function" and getsynasset)
    if type(getter) ~= "function" then return nil end

    local folder = "ItemManagementSystem"
    local path = folder .. "/IMS_Button.jpg"
    pcall(function()
        if type(isfolder) == "function" and type(makefolder) == "function" and not isfolder(folder) then
            makefolder(folder)
        end
    end)

    local exists = false
    pcall(function()
        exists = type(isfile) == "function" and isfile(path)
    end)
    if not exists then
        local okDecode, bytes = pcall(decodeIMSBase64, IMS_BUTTON_BASE64)
        if not okDecode or type(bytes) ~= "string" or #bytes == 0 then return nil end
        local okWrite = pcall(writefile, path, bytes)
        if not okWrite then return nil end
    end

    local okAsset, asset = pcall(getter, path)
    if okAsset and type(asset) == "string" and asset ~= "" then
        return asset
    end
    return nil
end

local IMS_BUTTON_ASSET = getIMSButtonAsset()

-- ========== ปุ่ม IMS ลอยสำหรับมือถือ (แตะครั้งเดียว เปิด/ปิด UI) ==========
do
    local menuGui, menuBtn
    local busy = false

    -- ใช้ Minimize ของ Fluent เป็นตัวสลับโดยตรง
    -- ไม่อ่าน Root.Visible เพราะ Fluent บางรุ่นซ่อนหน้าต่างด้วยวิธีที่สถานะ Visible ของ child ไม่ตรงกัน
    local function toggleWindow()
        if busy or Fluent.Unloaded then return end
        busy = true

        local ok = pcall(function()
            Window:Minimize()
        end)

        -- fallback เฉพาะกรณี Fluent ไม่มี Minimize / เรียกไม่ได้จริง
        if not ok then
            pcall(function()
                local gui = Fluent.GUI
                if typeof(gui) ~= "Instance" then return end
                local best, bestArea = nil, 0
                for _, d in ipairs(gui:GetDescendants()) do
                    if d:IsA("Frame") then
                        local a = d.AbsoluteSize.X * d.AbsoluteSize.Y
                        if a > bestArea then
                            best, bestArea = d, a
                        end
                    end
                end
                if best then best.Visible = not best.Visible end
            end)
        end

        task.delay(0.20, function() busy = false end)
    end

    if IS_MOBILE and not IMS_BUTTON_ASSET then
        notify("IMS", "executor นี้สร้างรูปปุ่มไม่ได้ จึงใช้ปุ่ม IMS สำรองแทน", 5)
    end

    local okMenu = pcall(function()
        menuGui = Instance.new("ScreenGui")
        menuGui.Name = "IMS_MenuButton"
        menuGui.ResetOnSpawn = false
        menuGui.IgnoreGuiInset = true
        menuGui.DisplayOrder = 10001
        menuGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
        menuGui.Enabled = IS_MOBILE
        menuGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

        menuBtn = Instance.new(IMS_BUTTON_ASSET and "ImageButton" or "TextButton")
        menuBtn.Name = "IMSButton"
        menuBtn.AnchorPoint = Vector2.new(0, 0.5)
        menuBtn.Size = UDim2.fromOffset(72, 72)
        menuBtn.Position = UDim2.new(0, 14, 0.34, 0)
        menuBtn.BackgroundColor3 = Color3.fromRGB(45, 55, 110)
        menuBtn.BackgroundTransparency = 0.04
        menuBtn.AutoButtonColor = true
        menuBtn.Active = true
        menuBtn.Selectable = true
        menuBtn.ClipsDescendants = true
        menuBtn.ZIndex = 20
        if menuBtn:IsA("ImageButton") then
            menuBtn.Image = IMS_BUTTON_ASSET
            menuBtn.ImageColor3 = Color3.new(1, 1, 1)
            menuBtn.ImageTransparency = 0
            menuBtn.ScaleType = Enum.ScaleType.Crop
        else
            menuBtn.TextColor3 = Color3.new(1, 1, 1)
            menuBtn.Font = Enum.Font.GothamBlack
            menuBtn.TextSize = 18
            menuBtn.Text = "IMS"
        end
        menuBtn.Parent = menuGui

        -- เพิ่มความสวยโดยยังคงรูปภาพเป็นตัวปุ่มหลัก
        local overlay = Instance.new("Frame")
        overlay.Name = "IMSOverlay"
        overlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        overlay.BackgroundTransparency = 0.72
        overlay.BorderSizePixel = 0
        overlay.Position = UDim2.new(0, 0, 0.72, 0)
        overlay.Size = UDim2.new(1, 0, 0.28, 0)
        overlay.ZIndex = 21
        overlay.Active = false
        overlay.Parent = menuBtn

        local title = Instance.new("TextLabel")
        title.BackgroundTransparency = 1
        title.Size = UDim2.new(1, -8, 1, 0)
        title.Position = UDim2.new(0, 4, 0, 0)
        title.Font = Enum.Font.GothamBlack
        title.TextSize = 10
        title.TextColor3 = Color3.new(1, 1, 1)
        title.TextStrokeTransparency = 0.25
        title.Text = "IMS"
        title.ZIndex = 22
        title.Parent = overlay

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 18)
        corner.Parent = menuBtn

        local stroke = Instance.new("UIStroke")
        stroke.Color = Color3.fromRGB(255, 255, 255)
        stroke.Transparency = 0.12
        stroke.Thickness = 2.5
        stroke.Parent = menuBtn

        -- ใช้ Activated อย่างเดียว: มือถือแตะครั้งเดียว / เมาส์คลิกครั้งเดียว ไม่เกิดอาการต้องกด 2-3 รอบ
        menuBtn.Activated:Connect(toggleWindow)
    end)
    if not okMenu then warn("[ItemSystem] สร้างปุ่ม IMS ไม่สำเร็จ") end

    local Sec = newSection(Tabs.General, "ปุ่มเปิด/ปิดเมนู")
    local Tg = Sec:AddToggle("MenuButton", {
        Title = "แสดงปุ่มเมนูลอยบนหน้าจอ",
        Description = "มือถือ: แตะรูป IMS ครั้งเดียวเพื่อเปิด/ปิด UI | คอมใช้ RightControl เหมือนเดิม",
        Default = IS_MOBILE,
    })
    Tg:OnChanged(function()
        if menuGui then
            menuGui.Enabled = IS_MOBILE and Options.MenuButton.Value
        end
    end)

    function Ext.menuCleanup()
        if menuGui then
            menuGui:Destroy()
            menuGui = nil
            menuBtn = nil
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
    pcall(Ext.menuCleanup)
    pcall(clearAllPlayerESP)
    pcall(stopVacuum)
    pcall(clearAllESP)
    pcall(setPerfLevel, 0)
    pcall(setFpsLock, false, FpsLock.Cap)
    if MobileWindowScaleConn then pcall(function() MobileWindowScaleConn:Disconnect() end); MobileWindowScaleConn = nil end
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
