local config = {
    name = "Blox Tools (Simulador Educativo)",
    auto_farm_enabled = true,
    auto_farm_interval = 2.0, -- segundos entre tentativas de farm
    farm_xp_min = 8,
    farm_xp_max = 15,
    auto_mastery_enabled = true,
    auto_mastery_check_interval = 5.0, -- segundos entre checagens de maestria
    auto_allocate_enabled = true,
    auto_allocate_interval = 1.0, -- segundos entre tentativas de alocar pontos
    allocate_priority = {"strength", "agility", "magic"}, -- prioridade de alocação
    max_ability_level = 50, -- nível máximo por habilidade (simulado)
    log_interval = 10.0, -- intervalos para exibir resumo no console
}

-- Simulação do jogador
local player = {
    level = 1,
    xp = 0,
    mastery = 0,
    ability_points = 0,
    abilities = {
        strength = 0,
        agility = 0,
        magic = 0,
    },
    gold = 0,
}

-- Funções utilitárias
local has_socket, socket = pcall(require, "socket")
local function sleep(sec)
    if has_socket and socket.sleep then
        socket.sleep(sec)
    else
        -- fallback simples (apenas segundos inteiros em Windows ou Unix)
        if package.config:sub(1,1) == "\\" then
            -- Windows
            os.execute("ping -n " .. tonumber(sec + 1) .. " localhost > nul")
        else
            os.execute("sleep " .. tonumber(sec))
        end
    end
end

local function rnd(minv, maxv)
    return math.floor(math.random() * (maxv - minv + 1)) + minv
end

-- XP necessário para o próximo nível (exemplo simples)
local function xp_for_next_level(level)
    -- fórmula simples: 100 * level
    return 100 * level
end

-- Nível up
local function try_level_up()
    local leveled = false
    while player.xp >= xp_for_next_level(player.level) do
        player.xp = player.xp - xp_for_next_level(player.level)
        player.level = player.level + 1
        player.ability_points = player.ability_points + 1 -- ganha 1 ponto por level
        leveled = true
        print((">>> Level up! Agora nível %d. Pontos de habilidade ganhos: 1 (total %d)"):format(player.level, player.ability_points))
    end
    return leveled
end

-- Checar maestria (exemplo simples: a cada 5 níveis => +1 maestria)
local function try_increase_mastery()
    local next_mastery_level = (player.mastery + 1) * 5
    if player.level >= next_mastery_level then
        player.mastery = player.mastery + 1
        -- recompensa de maestria (simulada)
        local reward_gold = 50 * player.mastery
        player.gold = player.gold + reward_gold
        print(("*** Maestria aumentada para %d! Recompensa: %d de ouro. Ouro total: %d"):format(player.mastery, reward_gold, player.gold))
        return true
    end
    return false
end

-- Simula uma ação de farm (derrotar inimigo/coletar recurso)
local function farm_once()
    -- Ganha XP aleatório e ouro aleatório
    local xp_gain = rnd(config.farm_xp_min, config.farm_xp_max)
    local gold_gain = rnd(1, 5)
    player.xp = player.xp + xp_gain
    player.gold = player.gold + gold_gain
    print(("Farm: +%d XP, +%d ouro (XP atual: %d, Ouro: %d)"):format(xp_gain, gold_gain, player.xp, player.gold))
    -- Tenta subir de nível
    try_level_up()
end

-- Alocação automática de pontos de habilidade baseada em prioridade
local function allocate_points()
    if player.ability_points <= 0 then return false end
    local allocated_any = false
    for _, ability in ipairs(config.allocate_priority) do
        while player.ability_points > 0 and player.abilities[ability] < config.max_ability_level do
            player.abilities[ability] = player.abilities[ability] + 1
            player.ability_points = player.ability_points - 1
            allocated_any = true
            print(("-> Alocado 1 ponto em %s (agora %d). Pontos restantes: %d"):format(ability, player.abilities[ability], player.ability_points))
            -- se quiser alocar somente 1 ponto por ciclo, descomente a linha a seguir:
            -- break
        end
        if player.ability_points <= 0 then break end
    end
    return allocated_any
end

-- Estado e loop principal
local function print_summary()
    print("=================================================")
    print(("Estado do jogador — Nível: %d | XP: %d/%d | Maestria: %d | Pontos: %d | Ouro: %d")
        :format(player.level, player.xp, xp_for_next_level(player.level), player.mastery, player.ability_points, player.gold))
    print(("Habilidades: strength=%d, agility=%d, magic=%d")
        :format(player.abilities.strength, player.abilities.agility, player.abilities.magic))
    print(("Config: auto_farm=%s, auto_mastery=%s, auto_allocate=%s")
        :format(tostring(config.auto_farm_enabled), tostring(config.auto_mastery_enabled), tostring(config.auto_allocate_enabled)))
    print("=================================================")
end

local function run_simulation(runtime_seconds)
    runtime_seconds = runtime_seconds or 60 * 60 -- default 1 hora se não especificado
    local start_time = os.time()
    local last_farm = 0
    local last_mastery = 0
    local last_allocate = 0
    local last_log = 0

    math.randomseed(os.time() % 4294967295)

    print(("Iniciando %s — simulação por %ds"):format(config.name, runtime_seconds))
    print_summary()

    while os.time() - start_time < runtime_seconds do
        local now = os.time()

        if config.auto_farm_enabled then
            if now - last_farm >= config.auto_farm_interval then
                farm_once()
                last_farm = now
            end
        end

        if config.auto_mastery_enabled then
            if now - last_mastery >= config.auto_mastery_check_interval then
                try_increase_mastery()
                last_mastery = now
            end
        end

        if config.auto_allocate_enabled then
            if now - last_allocate >= config.auto_allocate_interval then
                allocate_points()
                last_allocate = now
            end
        end

        if now - last_log >= config.log_interval then
            print_summary()
            last_log = now
        end

        -- dormir um pouco para não consumir CPU
        sleep(0.2)
    end

    print("Simulação finalizada.")
    print_summary()
end

-- Interface simples: você pode alterar config no topo antes de rodar ou chamar funções diretamente em um REPL.
-- Exemplo de uso:
--   lua blox_tools.lua
-- Para executar a simulação por 120 segundos, descomente e ajuste abaixo:
-- run_simulation(120)

-- Se você quiser rodar automaticamente por 2 minutos ao executar:
if ... == nil then
    -- script rodando sem argumentos: inicia simulação de exemplo (120s)
    run_simulation(120)
end

return {
    config = config,
    player = player,
    run_simulation = run_simulation,
    farm_once = farm_once,
    try_level_up = try_level_up,
    try_increase_mastery = try_increase_mastery,
    allocate_points = allocate_points,
    print_summary = print_summary,
}
