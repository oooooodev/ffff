--====================================================================
-- PAINEL v64 — FINAL OTIMIZADO
-- ====================================================================
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local Colors = {
    bg=Color3.fromRGB(15,15,20), panel=Color3.fromRGB(22,22,30), card=Color3.fromRGB(28,28,38),
    card2=Color3.fromRGB(34,34,46), border=Color3.fromRGB(45,45,60),
    accent=Color3.fromRGB(139,92,246), accent2=Color3.fromRGB(168,85,247),
    green=Color3.fromRGB(34,197,94), red=Color3.fromRGB(239,68,68),
    yellow=Color3.fromRGB(251,191,36), text=Color3.fromRGB(240,240,250),
    textDim=Color3.fromRGB(150,150,170), off=Color3.fromRGB(60,60,75),
}

local function FormatarNumero(n)
    if not n then return "0" end
    n = math.floor(n)
    local s = tostring(n)
    local resultado = ""
    local c = 0
    for i = #s, 1, -1 do
        resultado = s:sub(i, i) .. resultado
        c = c + 1
        if c % 3 == 0 and i > 1 then resultado = "." .. resultado end
    end
    return resultado
end

local POS_NPC_SELL = Vector3.new(274, 18, -66)
local POS_LAUNDER  = Vector3.new(6806, 18, -34)

local CONFIG = {
    Velocidade = 250, Chegada = 6, HoldETempo = 2.0,
    YFlutuacao = 22, YCarroDetectado = 38, YMin = 14, YForce = 9999999,
    TP_Steps = 8, TP_Delay = 0.03, TP_Repeticoes = 3,
    MaxVendaTentativas = 40, MaxFalhasSeguidas = 8,
    TravarFrente = false, DetectarCarro = true,
    UltraRapid = false, AntiBan = false, ChecarDinheiro = true,
    WatchdogSegundos = 120,
}

local function W(t, tRap)
    tRap = tRap or 0.05
    local v = CONFIG.UltraRapid and tRap or t
    if CONFIG.AntiBan and not CONFIG.UltraRapid then v = v * (0.7 + math.random()*0.6) end
    task.wait(v)
end
local function WR(t)
    if CONFIG.AntiBan then t = t * (0.85 + math.random()*0.3) end
    task.wait(t)
end

local YFlutuacaoAtual = CONFIG.YFlutuacao

local ULTIMA_LAVAGEM = os.clock()
local function ResetarWatchdog() ULTIMA_LAVAGEM = os.clock() end
local function SegundosSemLavar() return os.clock() - ULTIMA_LAVAGEM end

local function ResetarPersonagem()
    print("[WATCHDOG] Resetando personagem...")
    pcall(function()
        local ch = LocalPlayer.Character
        if ch then
            local hum = ch:FindFirstChildOfClass("Humanoid")
            if hum then hum.Health = 0 else ch:BreakJoints() end
        end
    end)
end

local ItensDisponiveis = {
    {nome = "Mona Lisa Painting",       pos = Vector3.new(6803.5, 18.4, 23.1), qtd = 5, ativo = false, preco = 3750},
    {nome = "Crate Of Avacados",        pos = Vector3.new(6820.8, 18.4, 37.4), qtd = 5, ativo = false, preco = 150},
    {nome = "Wagyu Beef",               pos = Vector3.new(6811.5, 18.4, 38.5), qtd = 5, ativo = false, preco = 350},
    {nome = "Witches Brew",             pos = Vector3.new(6803.7, 18.4, 33.4), qtd = 5, ativo = false, preco = 500},
    {nome = "Fake Designer Sneakers",   pos = Vector3.new(6810.0, 18.4, 17.0), qtd = 5, ativo = false, preco = 700},
    {nome = "Fake Diamond Ring",        pos = Vector3.new(6820.8, 18.4, 17.0), qtd = 5, ativo = false, preco = 1000},
}

local IGNORAR_VENDA = {
    "punhos","punho","fists","fist","golpe","punch",
    "telefone","phone","celular","mobile","cell",
    "mirtilo","mirttilo","blueberry","chave","key","keys","keycar",
    "carro","car","vehicle","mapa","map","gps","radio","walkie",
    "algema","handcuff","carteira","wallet",
    "briefcase","maleta","case","pasta","valise",
}

local function LimparNumero(s)
    if not s then return nil end
    s = tostring(s):lower():gsub("%s", "")
    local mult = 1
    if s:find("mil", 1, true) then mult = 1000; s = s:gsub("mil", "")
    elseif s:find("mi", 1, true) then mult = 1000000; s = s:gsub("mi", "")
    elseif s:find("k", 1, true) then mult = 1000; s = s:gsub("k", "") end
    if s:find(",") then
        local a, d = s:match("^(%d+),(%d+)$")
        if a and d and #d == 3 then s = s:gsub(",", "") else s = s:gsub(",", ".") end
    end
    if s:find("%.") then
        local a, d = s:match("^(%d+)%.(%d+)$")
        if a and d and #d == 3 then s = s:gsub("%.", "") end
    end
    local n = tonumber(s)
    if n then return math.floor(n * mult + 0.5) end
    return nil
end

local CACHE_DINHEIRO = {valor = nil, t = 0, bruto = nil}
local function PegarDinheiro()
    if os.clock() - CACHE_DINHEIRO.t < 0.3 then return CACHE_DINHEIRO.valor end
    local valor, bruto = nil, nil
    local rs = LocalPlayer:FindFirstChild("ReplicatedStats")
    if rs then
        local m = rs:FindFirstChild("Money")
        if m and (m:IsA("StringValue") or m:IsA("IntValue") or m:IsA("NumberValue")) then
            bruto = tostring(m.Value)
            valor = LimparNumero(m.Value)
        end
    end
    CACHE_DINHEIRO.valor = valor
    CACHE_DINHEIRO.bruto = bruto
    CACHE_DINHEIRO.t = os.clock()
    return valor
end
local function PegarDinheiroBruto() return CACHE_DINHEIRO.bruto end

local function ParsePreco(texto)
    if not texto or texto == "" then return nil end
    local limpo = tostring(texto):gsub(",", "")
    local v = limpo:match("%$(%d+)")
    if v then return tonumber(v) end
    return nil
end

local Rota = {
    Vector3.new(6935,16,0),Vector3.new(6935,16,2),Vector3.new(6935,16,5),
    Vector3.new(6935,16,8),Vector3.new(6935,16,11),Vector3.new(6934,16,15),
    Vector3.new(6934,16,18),Vector3.new(6934,16,21),Vector3.new(6934,16,24),
    Vector3.new(6934,16,28),Vector3.new(6934,16,33),Vector3.new(6933,16,37),
    Vector3.new(6933,16,43),Vector3.new(6933,16,48),Vector3.new(6933,16,54),
    Vector3.new(6933,16,60),Vector3.new(6932,16,66),Vector3.new(6932,16,71),
    Vector3.new(6932,16,76),Vector3.new(6932,16,81),Vector3.new(6931,16,86),
    Vector3.new(6931,16,91),Vector3.new(6929,16,96),Vector3.new(6927,16,99),
    Vector3.new(6924,16,103),Vector3.new(6921,16,105),Vector3.new(6917,16,107),
    Vector3.new(6913,16,109),Vector3.new(6910,16,110),Vector3.new(6907,16,110),
    Vector3.new(6904,16,110),Vector3.new(6901,16,109),Vector3.new(6898,16,109),
    Vector3.new(6895,16,109),Vector3.new(6892,16,109),Vector3.new(6890,16,109),
    Vector3.new(6888,16,109),Vector3.new(6884,16,109),Vector3.new(6881,16,108),
    Vector3.new(6878,16,108),Vector3.new(6875,16,108),Vector3.new(6873,16,108),
    Vector3.new(6871,16,108),Vector3.new(6867,16,108),Vector3.new(6865,16,108),
    Vector3.new(6862,16,108),Vector3.new(6858,16,108),Vector3.new(6854,16,107),
    Vector3.new(6850,16,107),Vector3.new(6847,16,107),Vector3.new(6843,16,107),
    Vector3.new(6840,16,107),Vector3.new(6838,16,107),Vector3.new(6834,16,107),
    Vector3.new(3401,16,108),Vector3.new(3399,16,108),Vector3.new(3395,16,108),
    Vector3.new(3392,16,108),Vector3.new(3389,16,108),Vector3.new(3385,16,108),
    Vector3.new(3382,16,108),Vector3.new(3378,16,107),Vector3.new(3373,16,107),
    Vector3.new(3368,16,107),Vector3.new(3363,16,107),Vector3.new(3358,16,107),
    Vector3.new(3353,16,107),Vector3.new(3347,16,107),Vector3.new(3341,16,107),
    Vector3.new(3335,16,107),Vector3.new(3328,16,107),Vector3.new(3321,16,107),
    Vector3.new(3315,16,107),Vector3.new(3308,16,107),Vector3.new(3301,16,107),
    Vector3.new(3294,16,107),Vector3.new(3286,16,107),Vector3.new(3278,16,106),
    Vector3.new(3270,16,106),Vector3.new(3262,16,106),Vector3.new(3252,16,106),
    Vector3.new(3244,16,106),Vector3.new(3235,16,106),Vector3.new(3227,16,106),
    Vector3.new(3196,16,106),Vector3.new(3193,16,106),Vector3.new(3189,16,106),
    Vector3.new(3187,16,106),Vector3.new(3183,16,106),Vector3.new(3179,16,106),
    Vector3.new(3175,16,106),Vector3.new(3171,16,106),Vector3.new(3166,16,106),
    Vector3.new(3160,16,106),Vector3.new(3154,16,106),Vector3.new(3148,16,106),
    Vector3.new(3141,16,106),Vector3.new(3133,16,106),Vector3.new(3125,16,106),
    Vector3.new(3117,16,106),Vector3.new(3110,16,106),Vector3.new(3104,16,106),
    Vector3.new(3099,16,106),Vector3.new(3094,16,106),Vector3.new(3088,16,106),
    Vector3.new(3083,16,106),Vector3.new(3077,16,106),Vector3.new(3072,16,106),
    Vector3.new(3066,16,106),Vector3.new(3061,16,106),Vector3.new(3055,16,106),
    Vector3.new(3049,16,106),Vector3.new(3044,16,107),Vector3.new(3038,16,107),
    Vector3.new(3033,16,107),Vector3.new(3027,16,107),Vector3.new(3022,16,108),
    Vector3.new(3016,16,108),Vector3.new(3011,16,108),Vector3.new(3005,16,108),
    Vector3.new(2999,16,108),Vector3.new(2994,16,108),Vector3.new(2988,16,108),
    Vector3.new(2982,16,108),Vector3.new(2976,16,108),Vector3.new(2971,16,108),
    Vector3.new(2965,16,108),Vector3.new(2959,16,108),Vector3.new(2954,16,108),
    Vector3.new(2948,16,108),Vector3.new(2943,16,108),Vector3.new(2937,16,108),
    Vector3.new(2932,16,108),Vector3.new(2926,16,108),Vector3.new(2920,16,108),
    Vector3.new(2915,16,108),Vector3.new(2909,16,108),Vector3.new(2903,16,108),
    Vector3.new(2898,16,109),Vector3.new(2892,16,110),Vector3.new(2888,16,113),
    Vector3.new(2884,16,116),Vector3.new(2880,16,118),Vector3.new(2876,16,120),
    Vector3.new(2872,16,121),Vector3.new(2868,16,123),Vector3.new(2863,16,124),
    Vector3.new(2858,16,126),Vector3.new(2853,16,128),Vector3.new(2848,16,129),
    Vector3.new(2842,16,129),Vector3.new(2838,16,129),Vector3.new(2834,16,129),
    Vector3.new(2830,16,128),Vector3.new(2827,16,128),Vector3.new(2823,16,128),
    Vector3.new(2819,16,128),Vector3.new(2814,16,128),Vector3.new(2809,16,128),
    Vector3.new(2804,16,128),Vector3.new(2783,16,128),Vector3.new(2780,16,128),
    Vector3.new(2776,16,128),Vector3.new(2773,16,128),Vector3.new(2768,16,128),
    Vector3.new(2764,16,128),Vector3.new(2759,16,127),Vector3.new(2754,16,127),
    Vector3.new(2748,16,127),Vector3.new(2743,16,125),Vector3.new(2738,16,124),
    Vector3.new(2732,16,122),Vector3.new(2727,16,121),Vector3.new(2722,16,120),
    Vector3.new(2717,16,118),Vector3.new(2711,16,117),Vector3.new(2706,16,116),
    Vector3.new(2701,16,114),Vector3.new(2696,16,114),Vector3.new(2690,16,113),
    Vector3.new(2685,16,112),Vector3.new(2681,16,111),Vector3.new(2675,16,111),
    Vector3.new(2670,16,110),Vector3.new(2666,16,110),Vector3.new(2662,16,110),
    Vector3.new(2658,16,111),Vector3.new(2654,16,111),Vector3.new(2650,16,111),
    Vector3.new(2646,16,111),Vector3.new(2643,16,111),Vector3.new(2640,16,112),
    Vector3.new(2637,16,112),Vector3.new(2634,16,112),Vector3.new(2631,16,112),
    Vector3.new(2627,16,112),Vector3.new(2623,16,112),Vector3.new(2619,16,112),
    Vector3.new(2614,16,113),Vector3.new(2608,16,113),Vector3.new(2603,16,113),
    Vector3.new(2597,16,113),Vector3.new(2592,16,113),Vector3.new(2587,16,113),
    Vector3.new(2583,16,114),Vector3.new(2578,16,114),Vector3.new(2573,16,114),
    Vector3.new(2566,16,114),Vector3.new(2562,16,114),Vector3.new(2556,16,114),
    Vector3.new(2550,16,114),Vector3.new(2545,16,115),Vector3.new(2539,16,115),
    Vector3.new(2534,16,115),Vector3.new(2529,16,115),Vector3.new(2524,16,115),
    Vector3.new(2518,16,115),Vector3.new(2513,16,115),Vector3.new(2508,16,115),
    Vector3.new(2503,16,115),Vector3.new(2498,16,115),Vector3.new(2493,16,115),
    Vector3.new(2488,16,115),Vector3.new(2483,16,115),Vector3.new(2478,16,114),
    Vector3.new(2472,16,114),Vector3.new(2467,16,114),Vector3.new(2462,16,114),
    Vector3.new(2456,16,114),Vector3.new(2450,16,114),Vector3.new(2442,16,114),
    Vector3.new(2435,16,113),Vector3.new(2427,16,113),Vector3.new(2419,16,113),
    Vector3.new(2411,16,113),Vector3.new(2404,16,112),Vector3.new(2396,16,111),
    Vector3.new(2389,16,110),Vector3.new(2382,16,110),Vector3.new(2375,16,110),
    Vector3.new(2367,16,110),Vector3.new(2361,16,109),Vector3.new(2353,16,109),
    Vector3.new(2346,16,109),Vector3.new(2339,16,109),Vector3.new(2333,16,109),
    Vector3.new(2326,16,108),Vector3.new(2320,16,108),Vector3.new(2313,16,108),
    Vector3.new(2307,16,108),Vector3.new(2302,16,108),Vector3.new(2297,16,108),
    Vector3.new(2291,16,108),Vector3.new(2286,16,108),Vector3.new(2280,16,108),
    Vector3.new(2276,16,108),Vector3.new(2271,16,108),Vector3.new(2267,16,108),
    Vector3.new(2262,16,108),
    Vector3.new(564,16,108),Vector3.new(561,16,108),Vector3.new(557,16,108),
    Vector3.new(554,16,108),Vector3.new(551,16,108),Vector3.new(548,16,108),
    Vector3.new(544,16,108),Vector3.new(539,16,108),Vector3.new(534,16,108),
    Vector3.new(528,16,108),Vector3.new(523,16,108),Vector3.new(516,16,108),
    Vector3.new(510,16,108),Vector3.new(502,16,108),Vector3.new(495,16,108),
    Vector3.new(486,16,108),Vector3.new(478,16,108),Vector3.new(468,16,108),
    Vector3.new(460,16,108),Vector3.new(451,16,108),Vector3.new(443,16,108),
    Vector3.new(433,16,108),Vector3.new(424,16,108),Vector3.new(415,16,108),
    Vector3.new(405,16,108),Vector3.new(396,16,109),Vector3.new(387,16,109),
    Vector3.new(379,16,109),Vector3.new(370,16,108),Vector3.new(362,16,105),
    Vector3.new(355,16,101),Vector3.new(349,16,95),Vector3.new(345,16,89),
    Vector3.new(341,16,83),Vector3.new(337,16,77),Vector3.new(335,16,72),
    Vector3.new(333,16,66),Vector3.new(333,16,60),Vector3.new(334,16,55),
    Vector3.new(334,16,51),Vector3.new(334,16,49),Vector3.new(334,16,46),
    Vector3.new(334,16,43),Vector3.new(334,16,41),Vector3.new(334,16,38),
    Vector3.new(334,16,35),Vector3.new(334,16,31),Vector3.new(334,16,28),
    Vector3.new(334,16,23),Vector3.new(334,16,18),Vector3.new(333,16,14),
    Vector3.new(333,16,8),Vector3.new(333,16,3),Vector3.new(332,16,-4),
    Vector3.new(331,16,-10),Vector3.new(330,16,-16),Vector3.new(328,16,-23),
    Vector3.new(326,16,-31),Vector3.new(323,16,-38),Vector3.new(319,16,-45),
    Vector3.new(315,16,-50),Vector3.new(309,16,-55),Vector3.new(304,16,-58),
    Vector3.new(298,16,-61),Vector3.new(292,16,-65),Vector3.new(288,16,-67),
    Vector3.new(285,16,-69),
}
local RotaInvertida = {}
for i = #Rota, 1, -1 do table.insert(RotaInvertida, Rota[i]) end

local Rodando = false
local BodyVel, BodyPos, BodyGyr
local CarroRef = nil
local PromptsDesabilitados = {}
local PromptsSitBloqueados = {}

local function Log(msg)
    print("[AUTO] " .. msg)
    if StatusLabel then StatusLabel.Text = msg end
end

local function PegarHRP()
    local ch = LocalPlayer.Character
    if not ch then return nil end
    return ch:FindFirstChild("HumanoidRootPart")
end
local function PegarSeat()
    local ch = LocalPlayer.Character
    if not ch then return nil end
    local hum = ch:FindFirstChildOfClass("Humanoid")
    if not hum or not hum.SeatPart then return nil end
    return hum.SeatPart
end
local function PegarHum()
    local ch = LocalPlayer.Character
    if not ch then return nil end
    return ch:FindFirstChildOfClass("Humanoid")
end

local function Normalizar(s)
    s = s:lower()
    s = s:gsub("[áàãâä]","a"):gsub("[éèêë]","e"):gsub("[íìîï]","i")
         :gsub("[óòõôö]","o"):gsub("[úùûü]","u"):gsub("ç","c"):gsub("[%s_%-]","")
    return s
end
local function EhIgnorado(nome)
    local n = Normalizar(nome)
    for _, ig in ipairs(IGNORAR_VENDA) do
        local nig = Normalizar(ig)
        if n == nig or n:find(nig, 1, true) then return true end
    end
    return false
end

local function ColetarTools()
    local tools = {}
    local function add(c)
        if not c then return end
        for _, v in ipairs(c:GetChildren()) do
            if v:IsA("Tool") then table.insert(tools, v)
            elseif v:IsA("Folder") or v:IsA("Model") then add(v) end
        end
    end
    add(LocalPlayer:FindFirstChild("Backpack"))
    add(LocalPlayer.Character)
    return tools
end
local function ContarItensVendaveis()
    local total, det = 0, {}
    for _, tool in ipairs(ColetarTools()) do
        if not EhIgnorado(tool.Name) then
            total = total + 1
            det[tool.Name] = (det[tool.Name] or 0) + 1
        end
    end
    return total, det
end
local function ContarBriefcases()
    local total, det = 0, {}
    for _, tool in ipairs(ColetarTools()) do
        local n = Normalizar(tool.Name)
        if n:find("briefcase",1,true) or n:find("maleta",1,true) or n:find("valise",1,true) then
            total = total + 1
            det[tool.Name] = (det[tool.Name] or 0) + 1
        end
    end
    return total, det
end
local function ContarItemEspecifico(itemInfo)
    local c = 0
    local nc = Normalizar(itemInfo.nome)
    for _, tool in ipairs(ColetarTools()) do
        local ni = Normalizar(tool.Name)
        if ni == nc or ni:find(nc,1,true) or nc:find(ni,1,true) then c = c + 1 end
    end
    return c
end

local function EhPromptSentar(prompt)
    local n = Normalizar(prompt.Name)
    if n:find("drive",1,true) or n:find("driver",1,true) then return false end
    if n:find("sit",1,true) or n:find("passenger",1,true) or n:find("enter",1,true) or n:find("board",1,true) then return true end
    return false
end
local function EhPromptCarro(prompt)
    if prompt.Parent and prompt.Parent:IsA("VehicleSeat") then return true end
    local p, depth = prompt.Parent, 0
    while p and p ~= workspace and depth < 5 do
        if p:IsA("VehicleSeat") then return true end
        p = p.Parent; depth = depth + 1
    end
    local n = Normalizar(prompt.Name)
    if n:find("vehicl",1,true) or n:find("seat",1,true) or n:find("drive",1,true) or n:find("sit",1,true) or n:find("car",1,true) then
        return true
    end
    return false
end
local function EhPromptsSitBloqueavel(prompt)
    if not prompt:IsA("ProximityPrompt") then return false end
    if not EhPromptSentar(prompt) then return false end
    if not EhPromptCarro(prompt) then return false end
    return true
end

local function BloquearTodosSits()
    local n = 0
    for _, obj in ipairs(workspace:GetDescendants()) do
        if EhPromptsSitBloqueavel(obj) then
            obj.Enabled = false
            PromptsSitBloqueados[obj] = true
            n = n + 1
        end
    end
    print("[SIT-BLOCK] "..n.." sits bloqueados")
end

workspace.DescendantAdded:Connect(function(obj)
    if not obj:IsA("ProximityPrompt") then return end
    task.defer(function()
        task.wait(0.1)
        if obj.Parent and EhPromptsSitBloqueavel(obj) then
            obj.Enabled = false
            PromptsSitBloqueados[obj] = true
        end
    end)
end)

local function DesabilitarPromptsCarro()
    PromptsDesabilitados = {}
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("ProximityPrompt") and EhPromptCarro(obj) then
            PromptsDesabilitados[obj] = obj.Enabled
            obj.Enabled = false
        end
    end
end
local function RestaurarPromptsCarro()
    for obj, estado in pairs(PromptsDesabilitados) do
        if obj and obj.Parent then
            if PromptsSitBloqueados[obj] or EhPromptsSitBloqueavel(obj) then
                obj.Enabled = false
                PromptsSitBloqueados[obj] = true
            else
                obj.Enabled = estado
            end
        end
    end
    PromptsDesabilitados = {}
end

local function TPsuave(hrp, destino, passos, delay)
    passos = passos or CONFIG.TP_Steps
    delay = delay or CONFIG.TP_Delay
    if CONFIG.UltraRapid then passos = 3; delay = 0.01 end
    local posOrigem = hrp.Position
    local diff = destino - posOrigem
    for i = 1, passos do
        local t = i / passos
        local novaPos = posOrigem + diff * t
        pcall(function() hrp.CFrame = CFrame.new(novaPos) end)
        task.wait(delay)
    end
end
local function TPjogador(pos)
    local hrp = PegarHRP()
    if not hrp then return false end
    local reps = CONFIG.UltraRapid and 1 or CONFIG.TP_Repeticoes
    for _ = 1, reps do TPsuave(hrp, pos) end
    return true
end

local function PararCarro()
    local seat = PegarSeat()
    if not seat then return end
    if BodyVel then BodyVel:Destroy() BodyVel = nil end
    if BodyPos then BodyPos:Destroy() BodyPos = nil end
    if BodyGyr then BodyGyr:Destroy() BodyGyr = nil end
    pcall(function()
        seat.AssemblyLinearVelocity = Vector3.zero
        seat.AssemblyAngularVelocity = Vector3.zero
        seat.Throttle=0; seat.ThrottleFloat=0; seat.Steer=0; seat.SteerFloat=0
    end)
    WR(0.4)
end
local function SairDoCarro()
    PararCarro()
    local hum = PegarHum()
    if hum and hum.Sit then
        hum.Sit = false
        WR(1.0)
        return true
    end
    return false
end

--=============================================================
-- 🔥 ACHAR CARRO — sempre o mais perto
--=============================================================
local function AcharCarro()
    local hrp = PegarHRP()
    if not hrp then return nil end

    -- Se o último que usei ainda existe e tá livre, usa
    if CarroRef and CarroRef.Parent then
        local s = CarroRef:FindFirstChildOfClass("VehicleSeat", true)
        if s and not s.Occupant then return {seat=s, model=CarroRef} end
    end

    -- Procura o VehicleSeat mais próximo
    local melhor, menor = nil, math.huge
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("VehicleSeat") and not obj.Occupant then
            local d = (obj.Position - hrp.Position).Magnitude
            if d < menor then
                menor = d
                -- Guarda o Model pai (subindo até achar)
                local model = obj:FindFirstAncestorOfClass("Model")
                melhor = {seat = obj, model = model or obj}
            end
        end
    end
    return melhor
end

--=============================================================
-- 🔥 ENTRAR NO CARRO — 3 métodos
--=============================================================
local function EntrarNoCarro()
    if PegarSeat() then return true end

    for tent = 1, 3 do
        local alvo = AcharCarro()
        if not alvo then
            print("[CARRO] Nenhum carro livre — tent "..tent.."/3")
            task.wait(1)
        else
            CarroRef = alvo.model
            local seatPos = alvo.seat.Position
            print("[CARRO] TP: "..alvo.model.Name.." ("..math.floor((seatPos - PegarHRP().Position).Magnitude).." studs)")
            TPjogador(seatPos)
            task.wait(0.6)

            -- Coleta prompts
            local prompts = {}
            for _, v in ipairs(alvo.seat:GetDescendants()) do
                if v:IsA("ProximityPrompt") then table.insert(prompts, v) end
            end
            if alvo.model ~= alvo.seat then
                for _, v in ipairs(alvo.model:GetDescendants()) do
                    if v:IsA("ProximityPrompt") and not table.find(prompts, v) then
                        table.insert(prompts, v)
                    end
                end
            end

            local promptDirigir, promptOutro = nil, nil
            for _, p in ipairs(prompts) do
                if EhPromptSentar(p) then
                    p.Enabled = false
                    PromptsSitBloqueados[p] = true
                elseif p.Parent and p.Parent:IsA("VehicleSeat") then
                    promptDirigir = p; break
                elseif not promptOutro then promptOutro = p end
            end

            local escolhido = promptDirigir or promptOutro

            -- Método 1: prompt
            if escolhido then
                pcall(function()
                    escolhido.Enabled = true
                    fireproximityprompt(escolhido)
                end)
                task.wait(0.8)
            end

            -- Método 2: seat:Sit
            if not PegarSeat() then
                local hum = PegarHum()
                if hum then
                    pcall(function() alvo.seat:Sit(hum) end)
                    task.wait(0.5)
                end
            end

            -- Método 3: TP + Sit
            if not PegarSeat() then
                local hum = PegarHum()
                local hrp = PegarHRP()
                if hum and hrp then
                    pcall(function()
                        hrp.CFrame = CFrame.new(seatPos + Vector3.new(0, 2, 0))
                        alvo.seat:Sit(hum)
                    end)
                    task.wait(0.8)
                end
            end

            if PegarSeat() then
                print("[CARRO] ✅ Entrou")
                return true
            end
        end
    end
    return false
end

local function IniciarFisica(seat)
    if BodyVel then BodyVel:Destroy() end
    if BodyPos then BodyPos:Destroy() end
    if BodyGyr then BodyGyr:Destroy() end
    pcall(function() seat.CustomPhysicalProperties = PhysicalProperties.new(3,0.5,0.5,1,1) end)
    if CarroRef then
        for _, v in ipairs(CarroRef:GetDescendants()) do
            if v:IsA("BasePart") and v ~= seat then
                pcall(function() v.Massless = true end)
            end
        end
    end
    BodyVel = Instance.new("BodyVelocity")
    BodyVel.MaxForce = Vector3.new(1e9,0,1e9); BodyVel.P = 5000
    BodyVel.Velocity = Vector3.zero; BodyVel.Parent = seat
    BodyPos = Instance.new("BodyPosition")
    BodyPos.MaxForce = Vector3.new(0,1e9,0); BodyPos.P = 150000; BodyPos.D = 15000
    BodyPos.Position = Vector3.new(seat.Position.X, math.max(seat.Position.Y, CONFIG.YMin), seat.Position.Z)
    BodyPos.Parent = seat
    BodyGyr = Instance.new("BodyGyro")
    BodyGyr.MaxTorque = Vector3.new(1e9,1e9,1e9); BodyGyr.P = 500000; BodyGyr.D = 50000
    BodyGyr.CFrame = seat.CFrame; BodyGyr.Parent = seat
end

local function DetectarCarroFrente(seat, direcao)
    if not CONFIG.DetectarCarro then return false end
    if not seat or not direcao then return false end
    local rp = RaycastParams.new()
    rp.FilterType = Enum.RaycastFilterType.Exclude
    rp.IgnoreWater = true
    local filtros = { LocalPlayer.Character }
    if CarroRef then table.insert(filtros, CarroRef) end
    rp.FilterDescendantsInstances = filtros
    local origens = {
        seat.Position,
        seat.Position + seat.CFrame.RightVector * 4,
        seat.Position - seat.CFrame.RightVector * 4,
    }
    for _, origem in ipairs(origens) do
        local result = workspace:Raycast(origem, direcao * 15, rp)
        if result and result.Instance then
            local p, depth = result.Instance, 0
            while p and p ~= workspace and depth < 6 do
                if p:FindFirstChildOfClass("VehicleSeat") then return true end
                p = p.Parent; depth = depth + 1
            end
        end
    end
    return false
end

--=============================================================
-- 🔥 RODAR ROTA (com anti-teleport / anti-stuck)
--=============================================================
local function RodarRota(rota, nome)
    local seat = PegarSeat()
    if not seat then
        print("[ROTA] Sem carro — abortando "..nome)
        return false
    end
    IniciarFisica(seat)
    YFlutuacaoAtual = CONFIG.YFlutuacao
    local idx = 1
    local ultimoYaw = nil
    local contCarro = 0
    local framesParado = 0
    local ultimaPos = seat.Position

    while idx <= #rota and Rodando do
        seat = PegarSeat()
        if not seat then
            print("[ROTA] Perdeu carro em "..nome)
            PararCarro()
            return false
        end

        local alvo = rota[idx]
        local pos = seat.Position
        local diff = Vector3.new(alvo.X-pos.X, 0, alvo.Z-pos.Z)
        local dist = diff.Magnitude
        local dirFrente = dist > 1 and diff.Unit or seat.CFrame.LookVector

        -- Anti-teleport
        if (pos - ultimaPos).Magnitude > 200 then
            print("[ROTA] Teleport detectado — abortando "..nome)
            PararCarro()
            return false
        end

        -- Anti-stuck
        if dist > CONFIG.Chegada and (pos - ultimaPos).Magnitude < 0.5 then
            framesParado = framesParado + 1
        else
            framesParado = 0
        end
        if framesParado > 600 then
            print("[ROTA] Travado em "..nome)
            PararCarro()
            return false
        end
        ultimaPos = pos

        if CONFIG.DetectarCarro then
            if DetectarCarroFrente(seat, dirFrente) then
                contCarro = math.min(contCarro + 1, 15)
            else
                contCarro = math.max(contCarro - 1, 0)
            end
        end
        local alvoY = CONFIG.YFlutuacao
        if contCarro > 3 then alvoY = CONFIG.YCarroDetectado end
        if alvoY < CONFIG.YMin then alvoY = CONFIG.YMin end
        YFlutuacaoAtual = YFlutuacaoAtual + (alvoY - YFlutuacaoAtual) * 0.12
        if YFlutuacaoAtual < CONFIG.YMin then YFlutuacaoAtual = CONFIG.YMin end
        if pos.Y < CONFIG.YMin - 1 then
            YFlutuacaoAtual = math.max(YFlutuacaoAtual, CONFIG.YMin + 4)
        end
        if BodyPos then BodyPos.Position = Vector3.new(pos.X, YFlutuacaoAtual, pos.Z) end
        local yaw
        if CONFIG.TravarFrente then
            if dist > 1 then yaw = math.atan2(-diff.X, -diff.Z); ultimoYaw = yaw
            elseif ultimoYaw then yaw = ultimoYaw
            else
                local look = seat.CFrame.LookVector
                yaw = math.atan2(-look.X, -look.Z); ultimoYaw = yaw
            end
        else
            local look = seat.CFrame.LookVector
            yaw = math.atan2(-look.X, -look.Z)
        end
        if BodyGyr then BodyGyr.CFrame = CFrame.new(pos) * CFrame.Angles(0, yaw, 0) end
        if CONFIG.TravarFrente then
            pcall(function()
                seat.CFrame = CFrame.new(pos.X, YFlutuacaoAtual, pos.Z) * CFrame.Angles(0, yaw, 0)
            end)
        end
        if dist < CONFIG.Chegada then
            idx = idx + 1
        else
            local vel = CONFIG.Velocidade / 3.6
            if BodyVel then
                BodyVel.Velocity = Vector3.new(dirFrente.X * vel, 0, dirFrente.Z * vel)
            end
        end
        RunService.Heartbeat:Wait()
    end
    PararCarro()
    return true
end

local function PegarPromptPerto(pos, raio, filtro)
    raio = raio or 30
    local melhor, menor = nil, math.huge
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("ProximityPrompt") and obj.Enabled then
            local match = true
            if filtro then match = obj.Name:lower():find(filtro:lower(),1,true) ~= nil end
            if match and not EhPromptCarro(obj) then
                local parent = obj.Parent
                local posP = nil
                if parent:IsA("BasePart") then posP = parent.Position
                elseif parent:IsA("Attachment") then posP = parent.WorldPosition end
                if posP then
                    local d = (posP - pos).Magnitude
                    if d < raio and d < menor then menor = d; melhor = obj end
                end
            end
        end
    end
    return melhor
end

--=============================================================
-- COMPRA
--=============================================================
local function ComprarItem(itemInfo)
    if not itemInfo.ativo then return "inativo" end
    local antes = ContarItemEspecifico(itemInfo)
    if antes >= itemInfo.qtd then return "completo" end

    if CONFIG.ChecarDinheiro then
        local dinheiro = PegarDinheiro()
        if dinheiro and itemInfo.preco and dinheiro < itemInfo.preco then
            return "sem_dinheiro"
        end
    end

    print("[COMPRA] "..itemInfo.nome.." — TP ("..antes.."/"..itemInfo.qtd..")")
    if PegarSeat() then SairDoCarro() end
    TPjogador(itemInfo.pos)
    task.wait(0.8)

    local t = 0
    local semPrompt = 0
    while t < 20 and Rodando do
        t = t + 1
        if ContarItemEspecifico(itemInfo) >= itemInfo.qtd then return "ok" end
        if CONFIG.ChecarDinheiro then
            local d = PegarDinheiro()
            if d and itemInfo.preco and d < itemInfo.preco then return "sem_dinheiro" end
        end

        local hrp = PegarHRP()
        if hrp then
            local prompt = PegarPromptPerto(hrp.Position, 30, "WorldBuyableItemPrompt")
            if prompt then
                semPrompt = 0
                pcall(function() fireproximityprompt(prompt) end)
                if keypress then pcall(function() keypress(0x45) end) end
                pcall(function() VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.E, false, game) end)
                task.wait(0.2)
                if keyrelease then pcall(function() keyrelease(0x45) end) end
                pcall(function() VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.E, false, game) end)
            else
                semPrompt = semPrompt + 1
                if semPrompt >= 3 then break end
                TPjogador(itemInfo.pos)
                task.wait(0.4)
            end
        end
        task.wait(0.4)
    end
    return "ok"
end

local function ComprarItens()
    local ativos = {}
    for _, item in ipairs(ItensDisponiveis) do
        if item.ativo then table.insert(ativos, item) end
    end
    if #ativos == 0 then return "ok" end

    local tudoCompleto = true
    for _, item in ipairs(ativos) do
        if ContarItemEspecifico(item) < item.qtd then tudoCompleto = false; break end
    end
    if tudoCompleto then return "ok" end

    if CONFIG.ChecarDinheiro then
        local dinheiro = PegarDinheiro()
        if dinheiro then
            local pode = false
            for _, item in ipairs(ativos) do
                if ContarItemEspecifico(item) < item.qtd then
                    if not item.preco or dinheiro >= item.preco then pode = true; break end
                end
            end
            if not pode then return "sem_dinheiro" end
        end
    end

    Log("Comprando...")
    for _, item in ipairs(ativos) do
        if not Rodando then break end
        ComprarItem(item)
        W(0.3, 0.05)
    end
    return "ok"
end

--=============================================================
-- SEGURAR E + VENDER
--=============================================================
local function SegurarE(tempo)
    tempo = tempo or CONFIG.HoldETempo
    if CONFIG.AntiBan then tempo = tempo * (0.9 + math.random() * 0.2) end
    if keypress then pcall(function() keypress(0x45) end) end
    pcall(function() VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.E, false, game) end)
    task.wait(tempo)
    if keyrelease then pcall(function() keyrelease(0x45) end) end
    pcall(function() VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.E, false, game) end)
end

local function Vender(posAlvo, filtro, nome, contadorFn, pularVoltarCarro, ehLaunder)
    contadorFn = contadorFn or ContarItensVendaveis
    pularVoltarCarro = pularVoltarCarro or false
    ehLaunder = ehLaunder or false

    PararCarro()
    WR(0.3)
    if PegarSeat() then SairDoCarro(); WR(0.5) end
    if posAlvo then TPjogador(posAlvo); WR(0.8) end

    local function Olhar()
        local hrp = PegarHRP()
        if hrp and posAlvo then
            pcall(function()
                local dir = posAlvo - hrp.Position
                local d = Vector3.new(dir.X, 0, dir.Z)
                if d.Magnitude > 0.1 then
                    hrp.CFrame = CFrame.lookAt(hrp.Position, hrp.Position + d.Unit)
                end
            end)
        end
    end
    Olhar()
    WR(0.3)

    DesabilitarPromptsCarro()
    WR(0.2)

    local totalVend = contadorFn()
    local totalBrief = ContarBriefcases()
    local cashAntes = PegarDinheiro() or 0
    print("ANTES ("..nome..") — V:"..totalVend.." B:"..totalBrief.." $:"..math.floor(cashAntes))

    if totalVend == 0 and totalBrief == 0 then
        Log("Nada ("..nome..")")
        WR(0.5)
        RestaurarPromptsCarro()
        if not pularVoltarCarro then
            if not EntrarNoCarro() then WR(2); EntrarNoCarro() end
        end
        return
    end

    Log(nome.." — "..totalVend)

    local t, falhas = 0, 0
    while t < CONFIG.MaxVendaTentativas and Rodando do
        t = t + 1
        for obj in pairs(PromptsDesabilitados) do
            if obj and obj.Parent and obj.Enabled then obj.Enabled = false end
        end

        local vAntes = contadorFn()
        local bAntes = ContarBriefcases()
        local dAntes = PegarDinheiro() or 0

        Olhar()
        WR(0.15)

        local hrp = PegarHRP()
        local promptAlvo = nil
        if hrp then promptAlvo = PegarPromptPerto(hrp.Position, 20, filtro) end

        if not promptAlvo then
            if posAlvo then TPjogador(posAlvo); WR(0.5); Olhar(); WR(0.2) end
            falhas = falhas + 1
            if falhas >= CONFIG.MaxFalhasSeguidas then break end
        else
            SegurarE(CONFIG.HoldETempo)
            task.wait(0.8)
        end

        local vDepois = contadorFn()
        local bDepois = ContarBriefcases()
        local dDepois = PegarDinheiro() or 0
        local vendeuItem = vDepois < vAntes
        local apareceuBrief = bDepois > bAntes
        local ganhouDinheiro = dDepois > (dAntes + 10)

        if vendeuItem or apareceuBrief or ganhouDinheiro then
            totalVend = vDepois
            falhas = 0
            if ehLaunder and ganhouDinheiro then ResetarWatchdog(); break end
            if vDepois == 0 then break end
        else
            falhas = falhas + 1
            if falhas >= CONFIG.MaxFalhasSeguidas then break end
            if falhas % 2 == 0 and posAlvo then TPjogador(posAlvo); WR(0.6)
            else Olhar(); WR(0.4) end
        end
    end

    Log("Fim ("..nome..")")
    WR(0.4)
    RestaurarPromptsCarro()
    WR(0.3)

    if not pularVoltarCarro then
        if not EntrarNoCarro() then WR(2); EntrarNoCarro() end
    end
end

local function TestarLaunder()
    task.spawn(function()
        print("🧪 TESTE LAUNDER")
        if PegarSeat() then SairDoCarro() end
        TPjogador(POS_LAUNDER)
        task.wait(1.2)
        local hrp = PegarHRP()
        if not hrp then return end
        pcall(function()
            local dir = POS_LAUNDER - hrp.Position
            dir = Vector3.new(dir.X, 0, dir.Z).Unit
            hrp.CFrame = CFrame.lookAt(hrp.Position, hrp.Position + dir)
        end)
        task.wait(0.3)
        local bAntes = ContarBriefcases()
        local cAntes = PegarDinheiro() or 0
        SegurarE(2)
        task.wait(2)
        local bDep = ContarBriefcases()
        local cDep = PegarDinheiro() or 0
        print("Brief: "..bAntes.." -> "..bDep)
        print("Cash: $"..FormatarNumero(cAntes).." -> $"..FormatarNumero(cDep))
        if cDep > cAntes or bDep < bAntes then
            print("✅ FUNCIONOU"); ResetarWatchdog()
        else
            print("⚠️ Nada")
        end
    end)
end

--=============================================================
-- 🔥 AUTOMAÇÃO OTIMIZADA — reset de estado por ciclo
--=============================================================
local function IniciarAutomacao()
    if Rodando then return end
    Rodando = true
    ResetarWatchdog()
    Log("Automação ON")
    task.spawn(function()
        local ciclo = 0
        while Rodando do
            ciclo = ciclo + 1
            print("")
            print("═══════ CICLO #"..ciclo.." ═══════")

            -- 🔥 RESET DE ESTADO — evita bug após muitos loops
            CarroRef = nil
            PromptsDesabilitados = {}
            if BodyVel then BodyVel:Destroy() BodyVel = nil end
            if BodyPos then BodyPos:Destroy() BodyPos = nil end
            if BodyGyr then BodyGyr:Destroy() BodyGyr = nil end

            Log("Comprando...")
            local resCompra = ComprarItens()
            if not Rodando then break end

            if not PegarSeat() then
                Log("Entrando...")
                if not EntrarNoCarro() then WR(3); continue end
            end
            if not Rodando then break end

            Log("IDA...")
            if not RodarRota(Rota, "IDA") then WR(2); continue end
            if not Rodando then break end

            Vender(POS_NPC_SELL, "SellSmuggledGoods", "NPC", ContarItensVendaveis, false, false)
            if not Rodando then break end

            if not PegarSeat() then
                if not EntrarNoCarro() then WR(2); continue end
            end
            if not Rodando then break end

            Log("VOLTA...")
            if not RodarRota(RotaInvertida, "VOLTA") then WR(2); continue end
            if not Rodando then break end

            Log("Lavando...")
            Vender(POS_LAUNDER, "LaunderBriefcasePrompt", "Launder", ContarBriefcases, true, true)
            if not Rodando then break end

            Log("Ciclo OK")
            WR(0.5)
        end
        PararCarro()
        Log("Parou")
    end)
end
local function PararAutomacao()
    Rodando = false
    PararCarro()
    RestaurarPromptsCarro()
    Log("Parando...")
end

task.spawn(function()
    task.wait(1)
    BloquearTodosSits()
    while true do
        task.wait(5)
        for _, obj in ipairs(workspace:GetDescendants()) do
            if obj:IsA("ProximityPrompt") and EhPromptsSitBloqueavel(obj) and obj.Enabled then
                obj.Enabled = false; PromptsSitBloqueados[obj] = true
            end
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(5)
        if Rodando then
            local s = SegundosSemLavar()
            if s >= CONFIG.WatchdogSegundos then
                print("[WATCHDOG] RESET")
                ResetarPersonagem()
                ResetarWatchdog()
                task.wait(2)
            end
        end
    end
end)

--=============================================================
-- GUI
--=============================================================
for _, n in ipairs({"PainelTrollMaster","FarmGui","FullControlGui","FazerTudo"}) do
    local old = PlayerGui:FindFirstChild(n)
    if old then old:Destroy() end
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "PainelTrollMaster"
ScreenGui.Parent = PlayerGui
ScreenGui.ResetOnSpawn = false
ScreenGui.DisplayOrder = 999

local Main = Instance.new("Frame")
Main.Size = UDim2.fromOffset(680, 460)
Main.Position = UDim2.new(0.5,-340,0.5,-230)
Main.BackgroundColor3 = Colors.bg
Main.BorderSizePixel = 0
Main.Active = true
Main.Parent = ScreenGui
Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 12)
local stroke = Instance.new("UIStroke", Main)
stroke.Color = Colors.border; stroke.Thickness = 1; stroke.Transparency = 0.3

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1,0,0,44)
Header.BackgroundColor3 = Colors.panel
Header.BorderSizePixel = 0
Header.Parent = Main
Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 12)
local HeaderFix = Instance.new("Frame")
HeaderFix.Size = UDim2.new(1,0,0,12)
HeaderFix.Position = UDim2.new(0,0,1,-12)
HeaderFix.BackgroundColor3 = Colors.panel
HeaderFix.BorderSizePixel = 0
HeaderFix.Parent = Header

local Logo = Instance.new("TextLabel")
Logo.Size = UDim2.fromOffset(200,44)
Logo.Position = UDim2.fromOffset(16,0)
Logo.BackgroundTransparency = 1
Logo.Text = "AIMTROLL FARM"
Logo.TextColor3 = Colors.text
Logo.Font = Enum.Font.GothamBold
Logo.TextSize = 15
Logo.TextXAlignment = Enum.TextXAlignment.Left
Logo.Parent = Header

local version = Instance.new("TextLabel")
version.Size = UDim2.fromOffset(70,20)
version.Position = UDim2.new(1,-180,0.5,-10)
version.BackgroundColor3 = Colors.card
version.Text = " v64 "
version.TextColor3 = Colors.textDim
version.Font = Enum.Font.GothamBold
version.TextSize = 10
version.Parent = Header
Instance.new("UICorner", version).CornerRadius = UDim.new(0, 4)

local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.fromOffset(28,28)
CloseBtn.Position = UDim2.new(1,-42,0.5,-14)
CloseBtn.BackgroundColor3 = Color3.fromRGB(60,20,30)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(255,130,130)
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.TextSize = 14
CloseBtn.BorderSizePixel = 0
CloseBtn.AutoButtonColor = false
CloseBtn.Parent = Header
Instance.new("UICorner", CloseBtn).CornerRadius = UDim.new(0, 6)
CloseBtn.MouseButton1Click:Connect(function()
    ScreenGui:Destroy()
    if Rodando then PararAutomacao() end
end)

local dg, ds, sp
Header.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then dg=true; ds=i.Position; sp=Main.Position end
end)
UserInputService.InputChanged:Connect(function(i)
    if dg and i.UserInputType == Enum.UserInputType.MouseMovement then
        local d = i.Position - ds
        Main.Position = UDim2.new(sp.X.Scale, sp.X.Offset+d.X, sp.Y.Scale, sp.Y.Offset+d.Y)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then dg=false end
end)

local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0,150,1,-60)
Sidebar.Position = UDim2.fromOffset(10,50)
Sidebar.BackgroundColor3 = Colors.panel
Sidebar.BorderSizePixel = 0
Sidebar.Parent = Main
Instance.new("UICorner", Sidebar).CornerRadius = UDim.new(0, 10)
local SideLayout = Instance.new("UIListLayout")
SideLayout.Padding = UDim.new(0,4); SideLayout.SortOrder = Enum.SortOrder.LayoutOrder
SideLayout.Parent = Sidebar
local SidePad = Instance.new("UIPadding")
SidePad.PaddingTop = UDim.new(0,8); SidePad.PaddingLeft = UDim.new(0,8); SidePad.PaddingRight = UDim.new(0,8)
SidePad.Parent = Sidebar

local Content = Instance.new("Frame")
Content.Size = UDim2.new(1,-175,1,-60)
Content.Position = UDim2.fromOffset(165,50)
Content.BackgroundColor3 = Colors.panel
Content.BorderSizePixel = 0
Content.Parent = Main
Instance.new("UICorner", Content).CornerRadius = UDim.new(0, 10)

local Pages = {}
local CurrentTab = nil

local function CriarPagina(nome)
    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.new(1,-20,1,-20)
    page.Position = UDim2.fromOffset(10,10)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = 3
    page.ScrollBarImageColor3 = Colors.accent
    page.CanvasSize = UDim2.new(0,0,0,0)
    page.AutomaticCanvasSize = Enum.AutomaticSize.Y
    page.Visible = false
    page.Parent = Content
    local l = Instance.new("UIListLayout")
    l.Padding = UDim.new(0,10); l.SortOrder = Enum.SortOrder.LayoutOrder
    l.Parent = page
    Pages[nome] = page
    return page
end
local function IrPara(nome)
    for n, p in pairs(Pages) do p.Visible = (n == nome) end
    CurrentTab = nome
end

local function Secao(parent, titulo)
    local s = Instance.new("Frame")
    s.Size = UDim2.new(1,0,0,0)
    s.BackgroundTransparency = 1
    s.AutomaticSize = Enum.AutomaticSize.Y
    s.Parent = parent
    local l = Instance.new("UIListLayout")
    l.Padding = UDim.new(0,8); l.SortOrder = Enum.SortOrder.LayoutOrder
    l.Parent = s
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,0,0,20)
    lbl.BackgroundTransparency = 1
    lbl.Text = " " .. titulo
    lbl.TextColor3 = Colors.textDim
    lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = s
    return s
end

local function Toggle(parent, nome, descricao, ini, cb)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1,0,0,50)
    row.BackgroundColor3 = Colors.card
    row.BorderSizePixel = 0; row.Parent = parent
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,-80,0,20); lbl.Position = UDim2.fromOffset(12,8)
    lbl.BackgroundTransparency = 1; lbl.Text = nome
    lbl.TextColor3 = Colors.text; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.Parent = row
    local desc = Instance.new("TextLabel")
    desc.Size = UDim2.new(1,-80,0,16); desc.Position = UDim2.fromOffset(12,27)
    desc.BackgroundTransparency = 1; desc.Text = descricao or ""
    desc.TextColor3 = Colors.textDim; desc.Font = Enum.Font.Gotham; desc.TextSize = 10
    desc.TextXAlignment = Enum.TextXAlignment.Left; desc.Parent = row
    local sw = Instance.new("TextButton")
    sw.Size = UDim2.fromOffset(44,22); sw.Position = UDim2.new(1,-56,0.5,-11)
    sw.BackgroundColor3 = ini and Colors.accent or Colors.off
    sw.Text = ""; sw.BorderSizePixel = 0; sw.AutoButtonColor = false; sw.Parent = row
    Instance.new("UICorner", sw).CornerRadius = UDim.new(1, 0)
    local knob = Instance.new("Frame")
    knob.Size = UDim2.fromOffset(18,18)
    knob.Position = ini and UDim2.new(1,-20,0.5,-9) or UDim2.new(0,2,0.5,-9)
    knob.BackgroundColor3 = Color3.new(1,1,1)
    knob.BorderSizePixel = 0; knob.Parent = sw
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)
    local st = ini
    sw.MouseButton1Click:Connect(function()
        st = not st
        TweenService:Create(sw, TweenInfo.new(0.2), {BackgroundColor3 = st and Colors.accent or Colors.off}):Play()
        TweenService:Create(knob, TweenInfo.new(0.2), {Position = st and UDim2.new(1,-20,0.5,-9) or UDim2.new(0,2,0.5,-9)}):Play()
        pcall(cb, st)
    end)
    return row
end

local function Slider(parent, nome, min, max, ini, cb)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1,0,0,62)
    row.BackgroundColor3 = Colors.card
    row.BorderSizePixel = 0; row.Parent = parent
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,-80,0,20); lbl.Position = UDim2.fromOffset(12,8)
    lbl.BackgroundTransparency = 1; lbl.Text = nome
    lbl.TextColor3 = Colors.text; lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.Parent = row
    local val = Instance.new("TextLabel")
    val.Size = UDim2.fromOffset(60,22); val.Position = UDim2.new(1,-70,0,8)
    val.BackgroundColor3 = Colors.card2; val.Text = tostring(ini)
    val.TextColor3 = Colors.accent; val.Font = Enum.Font.GothamBold; val.TextSize = 11
    val.Parent = row
    Instance.new("UICorner", val).CornerRadius = UDim.new(0, 5)
    local track = Instance.new("Frame")
    track.Size = UDim2.new(1,-24,0,5); track.Position = UDim2.fromOffset(12,44)
    track.BackgroundColor3 = Colors.off; track.BorderSizePixel = 0; track.Parent = row
    Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)
    local p0 = (ini - min) / (max - min)
    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(p0,0,1,0); fill.BackgroundColor3 = Colors.accent
    fill.BorderSizePixel = 0; fill.Parent = track
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
    local knob = Instance.new("Frame")
    knob.Size = UDim2.fromOffset(14,14); knob.AnchorPoint = Vector2.new(0.5,0.5)
    knob.Position = UDim2.new(p0,0,0.5,0); knob.BackgroundColor3 = Color3.new(1,1,1)
    knob.BorderSizePixel = 0; knob.Parent = track
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)
    local hit = Instance.new("TextButton")
    hit.Size = UDim2.new(1,0,0,30); hit.Position = UDim2.new(0,0,1,-30)
    hit.BackgroundTransparency = 1; hit.Text = ""; hit.AutoButtonColor = false; hit.Parent = row
    local arr = false
    local function Up(x)
        local p = math.clamp((x - track.AbsolutePosition.X) / math.max(track.AbsoluteSize.X,1), 0, 1)
        local v = math.floor(min + (max - min) * p + 0.5)
        fill.Size = UDim2.new(p,0,1,0)
        knob.Position = UDim2.new(p,0,0.5,0)
        val.Text = tostring(v)
        pcall(cb, v)
    end
    hit.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then arr = true; Up(i.Position.X) end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if arr and i.UserInputType == Enum.UserInputType.MouseMovement then Up(i.Position.X) end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then arr = false end
    end)
    return row
end

local function Botao(parent, nome, cor, cb, altura)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1,0,0,altura or 40)
    b.BackgroundColor3 = cor
    b.Text = nome; b.TextColor3 = Color3.new(1,1,1)
    b.Font = Enum.Font.GothamBold; b.TextSize = 13
    b.BorderSizePixel = 0; b.AutoButtonColor = false; b.Parent = parent
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 8)
    b.MouseButton1Click:Connect(function()
        local ok, err = pcall(cb)
        if not ok then warn("[UI] "..tostring(err)) end
    end)
    return b
end

local function ItemRow(parent, item)
    local row = Instance.new("TextButton")
    row.Size = UDim2.new(1,0,0,46)
    row.BackgroundColor3 = Colors.card
    row.Text = ""; row.BorderSizePixel = 0; row.AutoButtonColor = false
    row.Parent = parent
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)
    local box = Instance.new("Frame")
    box.Size = UDim2.fromOffset(20,20); box.Position = UDim2.fromOffset(12,13)
    box.BackgroundColor3 = item.ativo and Colors.accent or Colors.off
    box.BorderSizePixel = 0; box.Parent = row
    Instance.new("UICorner", box).CornerRadius = UDim.new(0, 5)
    local check = Instance.new("TextLabel")
    check.Size = UDim2.new(1,0,1,0); check.BackgroundTransparency = 1
    check.Text = item.ativo and "v" or ""
    check.TextColor3 = Color3.new(1,1,1); check.TextSize = 14
    check.Font = Enum.Font.GothamBold; check.Parent = box
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,-130,0,16); lbl.Position = UDim2.fromOffset(42,6)
    lbl.BackgroundTransparency = 1; lbl.Text = item.nome
    lbl.TextColor3 = item.ativo and Colors.text or Colors.textDim
    lbl.Font = Enum.Font.GothamBold; lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.Parent = row
    local precoLbl = Instance.new("TextLabel")
    precoLbl.Size = UDim2.new(1,-130,0,14); precoLbl.Position = UDim2.fromOffset(42,24)
    precoLbl.BackgroundTransparency = 1
    precoLbl.Text = item.preco and ("$"..FormatarNumero(item.preco)) or ""
    precoLbl.TextColor3 = Colors.yellow; precoLbl.Font = Enum.Font.Gotham; precoLbl.TextSize = 10
    precoLbl.TextXAlignment = Enum.TextXAlignment.Left; precoLbl.Parent = row
    local cont = Instance.new("TextLabel")
    cont.Size = UDim2.fromOffset(56,22); cont.Position = UDim2.new(1,-66,0.5,-11)
    cont.BackgroundColor3 = Colors.card2; cont.Text = "0/"..item.qtd
    cont.TextColor3 = Colors.accent; cont.Font = Enum.Font.GothamBold; cont.TextSize = 11
    cont.Parent = row
    Instance.new("UICorner", cont).CornerRadius = UDim.new(0, 5)
    item._ui = {row=row, box=box, check=check, lbl=lbl, cont=cont}
    row.MouseButton1Click:Connect(function()
        item.ativo = not item.ativo
        if item.ativo then
            box.BackgroundColor3 = Colors.accent
            lbl.TextColor3 = Colors.text
            check.Text = "v"
        else
            box.BackgroundColor3 = Colors.off
            lbl.TextColor3 = Colors.textDim
            check.Text = ""
        end
    end)
end

CriarPagina("Home"); CriarPagina("Items"); CriarPagina("Config"); CriarPagina("Extra")

local abas = {
    {nome="Home"}, {nome="Items"}, {nome="Config"}, {nome="Extra"},
}
local sideBtns = {}
for _, a in ipairs(abas) do
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1,0,0,38)
    b.BackgroundColor3 = Colors.card; b.BackgroundTransparency = 1
    b.Text = "  "..a.nome
    b.TextColor3 = Colors.textDim; b.Font = Enum.Font.GothamBold; b.TextSize = 12
    b.TextXAlignment = Enum.TextXAlignment.Left; b.BorderSizePixel = 0
    b.AutoButtonColor = false; b.Parent = Sidebar
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 8)
    b.MouseButton1Click:Connect(function()
        for n, btn in pairs(sideBtns) do
            if n == a.nome then
                btn.BackgroundTransparency = 0
                btn.BackgroundColor3 = Colors.accent
                btn.TextColor3 = Color3.new(1,1,1)
            else
                btn.BackgroundTransparency = 1
                btn.TextColor3 = Colors.textDim
            end
        end
        IrPara(a.nome)
    end)
    sideBtns[a.nome] = b
end

IrPara("Home")
sideBtns["Home"].BackgroundTransparency = 0
sideBtns["Home"].BackgroundColor3 = Colors.accent
sideBtns["Home"].TextColor3 = Color3.new(1,1,1)

local homePage = Pages["Home"]

local moneyCard = Instance.new("Frame")
moneyCard.Size = UDim2.new(1,0,0,50)
moneyCard.BackgroundColor3 = Colors.card
moneyCard.BorderSizePixel = 0
moneyCard.Parent = homePage
Instance.new("UICorner", moneyCard).CornerRadius = UDim.new(0, 8)

MoneyLabel = Instance.new("TextLabel")
MoneyLabel.Size = UDim2.new(1,-150,1,0)
MoneyLabel.Position = UDim2.fromOffset(12,0)
MoneyLabel.BackgroundTransparency = 1
MoneyLabel.Text = "$ --"
MoneyLabel.TextColor3 = Colors.green
MoneyLabel.Font = Enum.Font.GothamBold
MoneyLabel.TextSize = 16
MoneyLabel.TextXAlignment = Enum.TextXAlignment.Left
MoneyLabel.Parent = moneyCard

local moneyRawLabel = Instance.new("TextLabel")
moneyRawLabel.Size = UDim2.fromOffset(140,20)
moneyRawLabel.Position = UDim2.new(1,-150,0.5,-10)
moneyRawLabel.BackgroundTransparency = 1
moneyRawLabel.Text = ""
moneyRawLabel.TextColor3 = Colors.textDim
moneyRawLabel.Font = Enum.Font.Gotham
moneyRawLabel.TextSize = 10
moneyRawLabel.TextXAlignment = Enum.TextXAlignment.Right
moneyRawLabel.Parent = moneyCard

local statusCard = Instance.new("Frame")
statusCard.Size = UDim2.new(1,0,0,60)
statusCard.BackgroundColor3 = Colors.card
statusCard.BorderSizePixel = 0
statusCard.Parent = homePage
Instance.new("UICorner", statusCard).CornerRadius = UDim.new(0, 8)

StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1,-20,1,0)
StatusLabel.Position = UDim2.fromOffset(12,0)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Pronto"
StatusLabel.TextColor3 = Colors.green
StatusLabel.Font = Enum.Font.GothamMedium
StatusLabel.TextSize = 12
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.Parent = statusCard

local invCard = Instance.new("Frame")
invCard.Size = UDim2.new(1,0,0,44)
invCard.BackgroundColor3 = Colors.card
invCard.BorderSizePixel = 0; invCard.Parent = homePage
Instance.new("UICorner", invCard).CornerRadius = UDim.new(0, 8)
InvLabel = Instance.new("TextLabel")
InvLabel.Size = UDim2.new(1,-20,1,0)
InvLabel.Position = UDim2.fromOffset(12,0)
InvLabel.BackgroundTransparency = 1
InvLabel.Text = "Vend: 0 | Brief: 0"
InvLabel.TextColor3 = Colors.green
InvLabel.Font = Enum.Font.GothamBold
InvLabel.TextSize = 12
InvLabel.TextXAlignment = Enum.TextXAlignment.Left
InvLabel.Parent = invCard

local wdCard = Instance.new("Frame")
wdCard.Size = UDim2.new(1,0,0,30)
wdCard.BackgroundColor3 = Colors.card2
wdCard.BorderSizePixel = 0; wdCard.Parent = homePage
Instance.new("UICorner", wdCard).CornerRadius = UDim.new(0, 8)
WatchdogLabel = Instance.new("TextLabel")
WatchdogLabel.Size = UDim2.new(1,-20,1,0)
WatchdogLabel.Position = UDim2.fromOffset(12,0)
WatchdogLabel.BackgroundTransparency = 1
WatchdogLabel.Text = "Watchdog: --"
WatchdogLabel.TextColor3 = Colors.yellow
WatchdogLabel.Font = Enum.Font.GothamBold
WatchdogLabel.TextSize = 11
WatchdogLabel.TextXAlignment = Enum.TextXAlignment.Left
WatchdogLabel.Parent = wdCard

local secMain = Secao(homePage, "CONTROLE")
BtnAuto = Botao(secMain, "LIGAR (L)", Colors.green, function()
    if Rodando then PararAutomacao() else IniciarAutomacao() end
    if BtnAuto then
        if Rodando then
            BtnAuto.Text = "PARAR (L)"
            BtnAuto.BackgroundColor3 = Colors.red
        else
            BtnAuto.Text = "LIGAR (L)"
            BtnAuto.BackgroundColor3 = Colors.green
        end
    end
end, 46)

local secModo = Secao(homePage, "MODO")
Toggle(secModo, "Ultra Rapid", "Remove delays", CONFIG.UltraRapid, function(v) CONFIG.UltraRapid = v end)
Toggle(secModo, "Anti-Ban", "Delays aleatorios", CONFIG.AntiBan, function(v) CONFIG.AntiBan = v end)
Toggle(secModo, "Checar Dinheiro", "Pula se sem grana", CONFIG.ChecarDinheiro, function(v) CONFIG.ChecarDinheiro = v end)

local itemsPage = Pages["Items"]
local secItems = Secao(itemsPage, "ITENS")
for _, item in ipairs(ItensDisponiveis) do ItemRow(secItems, item) end

local configPage = Pages["Config"]
local secVoo = Secao(configPage, "VOO")
Slider(secVoo, "Velocidade", 50, 2000, CONFIG.Velocidade, function(v) CONFIG.Velocidade = v end)
Slider(secVoo, "Altura Voo", 10, 80, CONFIG.YFlutuacao, function(v)
    CONFIG.YFlutuacao = v
    if CONFIG.YMin > v then CONFIG.YMin = math.max(10, v - 8) end
end)
Slider(secVoo, "Altura Carro Frente", 20, 100, CONFIG.YCarroDetectado, function(v) CONFIG.YCarroDetectado = v end)
Slider(secVoo, "Altura Minima", 5, 40, CONFIG.YMin, function(v) CONFIG.YMin = v end)
Slider(secVoo, "Watchdog (seg)", 30, 300, CONFIG.WatchdogSegundos, function(v) CONFIG.WatchdogSegundos = v end)

local extraPage = Pages["Extra"]
local secComp = Secao(extraPage, "COMPORTAMENTO")
Toggle(secComp, "Travar Frente", "Trava yaw", CONFIG.TravarFrente, function(v) CONFIG.TravarFrente = v end)
Toggle(secComp, "Desviar Carro", "Sobe antes de bater", CONFIG.DetectarCarro, function(v) CONFIG.DetectarCarro = v end)

local secTest = Secao(extraPage, "TESTES")
Botao(secTest, "🚗 ENTRAR NO CARRO", Colors.green, function()
    task.spawn(function() EntrarNoCarro() end)
end, 40)

Botao(secTest, "🗣️ TESTAR VENDA", Colors.accent2, function()
    Vender(POS_NPC_SELL, "SellSmuggledGoods", "TESTE", ContarItensVendaveis, false, false)
end, 36)

Botao(secTest, "💰 TESTAR LAUNDER", Color3.fromRGB(234, 179, 8), TestarLaunder, 36)

task.spawn(function()
    while ScreenGui.Parent do
        task.wait(0.4)
        local vendaveis = ContarItensVendaveis()
        local briefs = ContarBriefcases()
        if InvLabel then
            InvLabel.Text = "Vend: "..vendaveis.." | Brief: "..briefs
            InvLabel.TextColor3 = (vendaveis == 0 and briefs == 0) and Colors.red or Colors.green
        end
        if MoneyLabel then
            local d = PegarDinheiro()
            local bruto = PegarDinheiroBruto()
            if d then
                MoneyLabel.Text = "$ "..FormatarNumero(d)
                MoneyLabel.TextColor3 = Colors.green
            else
                MoneyLabel.Text = "$ -- (nao detectado)"
                MoneyLabel.TextColor3 = Colors.yellow
            end
            if moneyRawLabel then
                moneyRawLabel.Text = "bruto: '"..tostring(bruto).."'"
            end
        end
        if WatchdogLabel then
            if Rodando then
                local s = math.floor(SegundosSemLavar())
                local resta = math.max(0, CONFIG.WatchdogSegundos - s)
                WatchdogLabel.Text = "Watchdog: "..s.."s / "..CONFIG.WatchdogSegundos.."s (reset em "..resta.."s)"
                WatchdogLabel.TextColor3 = s > CONFIG.WatchdogSegundos * 0.7 and Colors.red or Colors.yellow
            else
                WatchdogLabel.Text = "Watchdog: desligado"
                WatchdogLabel.TextColor3 = Colors.textDim
            end
        end
        for _, item in ipairs(ItensDisponiveis) do
            if item._ui and item._ui.cont then
                local c = ContarItemEspecifico(item)
                item._ui.cont.Text = c.."/"..item.qtd
                item._ui.cont.TextColor3 = (c >= item.qtd and item.ativo) and Colors.green or Colors.accent
            end
        end
    end
end)

UserInputService.InputBegan:Connect(function(input, gp)
    if gp or UserInputService:GetFocusedTextBox() then return end
    if input.KeyCode == Enum.KeyCode.L then
        if Rodando then PararAutomacao() else IniciarAutomacao() end
        if BtnAuto then
            if Rodando then
                BtnAuto.Text = "PARAR (L)"
                BtnAuto.BackgroundColor3 = Colors.red
            else
                BtnAuto.Text = "LIGAR (L)"
                BtnAuto.BackgroundColor3 = Colors.green
            end
        end
    end
end)

print("v64 — OK | Carro mais perto | Reset de estado por ciclo")
