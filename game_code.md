-- lua

-- coin collector simulation
-- constants
local r = 2 -- particle radius
local min_dist = r * 2
local min_dist_sq = min_dist * min_dist
local cell_size = 8
local cols = 128 / cell_size
local rows = 128 / cell_size

-- physics params
local gravity_strength = 4  -- reduced from 20
local grab_radius = 30
local grab_radius_sq = grab_radius * grab_radius
local grab_speed = 4
local grab_snap_dist = 1
local damp = 0.9

-- cooldowns and usage timers (in frames at 60fps)
local left_cooldown = 0  -- current cooldown timer for left click
local right_cooldown = 0 -- current cooldown timer for right click
local left_cooldown_duration = 60  -- 1 second (60 frames)
local right_cooldown_duration = 180 -- 3 seconds (180 frames)

-- usage duration tracking
local left_usage_timer = 0  -- tracks how long left click has been held
local right_usage_timer = 0 -- tracks how long right click has been held
local max_usage_duration = 300 -- 5 seconds (300 frames)

-- previous mouse state for detecting releases
local prev_mouse_btn = 0

-- game objects
local particles = {}
local coins = {}
local obstacles = {} -- renamed from dangers
local grid = {}
for i=1,cols*rows do grid[i] = {} end

-- obstacle spawn timer
local obstacle_spawn_timer = 300 -- 5 seconds (300 frames)

-- game state
local game_over = false

-- mouse state
local grabbed_particle, is_grabbing
local mx, my, mouse_btn

-- precompute neighbors
local neighbors = {
    {1,0}, {0,1}, {1,1}, {-1,1} -- right, down, down-right, down-left
}

-- color palette
local colors = {
    bg = 1,   -- dark blue
    red = 8,
    orange = 9,
    yellow = 10,
    blue = 12,
    white = 15
}

local game_state = "playing" -- can be "playing", "countdown", or "game_over"
local countdown = 3
local countdown_timer = 0
local countdown_duration = 60 -- frames (1 second at 60fps) - changed from 30 to 60
local countdown_scale = 1
local countdown_colors = {8, 9, 10} -- red, orange, yellow for 3,2,1

-- reset_game function
function reset_game()
    -- reset particles
    particles = {
        {
            x = 64,
            y = 64,
            vx = 0,
            vy = 0,
            m = 1,
            col = colors.blue
        }
    }
    
    -- reset coins (only 1 coin)
    coins = {}
    add(coins, {
        x = rnd(128),
        y = rnd(128),
        collected = false
    })
    
    -- reset obstacles
    obstacles = {}
    
    -- reset timers and cooldowns
    left_cooldown = 0
    right_cooldown = 0
    left_usage_timer = 0
    right_usage_timer = 0
    obstacle_spawn_timer = 300
    
    -- reset mouse state
    grabbed_particle = nil
    is_grabbing = false
    prev_mouse_btn = 0
    
    -- reset game state
    game_state = "playing"
    countdown = 3
    countdown_timer = 0
end

function _init()
    poke(0x5f2d, 1) -- enable mouse
    reset_game()
end

function _update60()
    if game_state == "game_over" then
        if mouse_btn > 0 then
            start_restart_countdown()
        end
        return
    elseif game_state == "countdown" then
        countdown_timer += 1
        countdown_scale = 1 + abs(sin(time()*0.1))*0.5 -- pulse animation
        
        if countdown_timer >= countdown_duration then
            countdown -= 1
            countdown_timer = 0
            if countdown <= 0 then
                reset_game() -- reset the game after countdown
                return
            end
        end
        return -- skip other updates during countdown
    end
    
    -- update cooldowns
    if left_cooldown > 0 then left_cooldown -= 1 end
    if right_cooldown > 0 then right_cooldown -= 1 end
    
    -- get mouse state
    mx, my = stat(32) or 64, stat(33) or 64
    mouse_btn = stat(34)
    
    -- detect mouse button releases and start cooldowns
    if prev_mouse_btn == 1 and mouse_btn ~= 1 and left_usage_timer > 0 then
        -- left click was released, start cooldown
        left_cooldown = left_cooldown_duration
        left_usage_timer = 0
    end
    
    if prev_mouse_btn == 2 and mouse_btn ~= 2 and right_usage_timer > 0 then
        -- right click was released, start cooldown
        right_cooldown = right_cooldown_duration
        right_usage_timer = 0
    end
    
    prev_mouse_btn = mouse_btn
    
    -- update obstacle spawn timer (delete existing and spawn new every 5 seconds)
    obstacle_spawn_timer -= 1
    if obstacle_spawn_timer <= 0 then
        -- delete existing obstacles and spawn new one
        obstacles = {}
        add(obstacles, {
            x = rnd(128),
            y = rnd(128)
        })
        obstacle_spawn_timer = 300 -- reset timer for next spawn
    end

    -- handle grabbing with usage timer and cooldown
    if mouse_btn == 2 and right_cooldown <= 0 then
        -- check if we can still use right click (within 5 second limit)
        if right_usage_timer < max_usage_duration then
            right_usage_timer += 1
            
            if not grabbed_particle then
                -- find closest particle
                local p = particles[1]
                local dx, dy = mx - p.x, my - p.y
                local d2 = dx*dx + dy*dy
                
                if d2 < grab_radius_sq then
                    grabbed_particle = p
                    p.vx, p.vy = 0, 0
                    is_grabbing = false
                end
            else
                -- move toward cursor
                local dx, dy = mx - grabbed_particle.x, my - grabbed_particle.y
                local dist = sqrt(dx*dx + dy*dy)
                
                if dist > grab_snap_dist then
                    is_grabbing = false
                    grabbed_particle.vx = dx/dist * grab_speed
                    grabbed_particle.vy = dy/dist * grab_speed
                else
                    is_grabbing = true
                    grabbed_particle.x, grabbed_particle.y = mx, my
                    grabbed_particle.vx, grabbed_particle.vy = 0, 0
                end
            end
        end
    elseif mouse_btn ~= 2 then
        grabbed_particle, is_grabbing = nil, false
    end

    -- update physics
    for p in all(particles) do
        if p != grabbed_particle or not is_grabbing then
            -- no gravity applied
            
            if mouse_btn == 1 and left_cooldown <= 0 and left_usage_timer < max_usage_duration then -- left click attraction with usage limit
                left_usage_timer += 1
                local dx, dy = mx - p.x, my - p.y
                local f = gravity_strength/(dx*dx + dy*dy + 0.1)
                p.vx += dx * f
                p.vy += dy * f
            end

            -- update position
            p.x += p.vx
            p.y += p.vy

            -- wall collisions
            if p.x < r then p.x, p.vx = r, -p.vx*damp end
            if p.x > 128-r then p.x, p.vx = 128-r, -p.vx*damp end
            if p.y < r then p.y, p.vy = r, -p.vy*damp end
            if p.y > 128-r then p.y, p.vy = 128-r, -p.vy*damp end

            -- coin collection check
            for c in all(coins) do
                if not c.collected and sqrt((p.x-c.x)^2 + (p.y-c.y)^2) < r+4 then
                    c.collected = true
                    -- spawn new coin (only 1 at a time)
                    add(coins, {
                        x = rnd(128),
                        y = rnd(128),
                        collected = false
                    })
                end
            end
            
            -- obstacle collision check - start countdown instead of immediate game over
            for o in all(obstacles) do
                if sqrt((p.x-o.x)^2 + (p.y-o.y)^2) < r+4 then
                    -- player touched obstacle - start countdown
                    game_state = "countdown"
                    countdown = 3
                    countdown_timer = 0
                    return -- exit update loop since countdown started
                end
            end
        end
    end

    -- remove collected coins
    for i=#coins,1,-1 do
        if coins[i].collected then
            del(coins, coins[i])
        end
    end

    -- rebuild spatial grid (for potential future particles)
    for i=1,#grid do grid[i] = {} end
    for p in all(particles) do
        local idx = flr(p.y/cell_size)*cols + flr(p.x/cell_size) + 1
        if idx >= 1 and idx <= #grid then
            add(grid[idx], p)
        end
    end
end

function start_restart_countdown()
    game_state = "countdown"
    countdown = 3
    countdown_timer = 0
end


function _draw()
    cls(colors.bg)
    
    if game_state == "game_over" then
        -- center game over text
        local text = "game over"
        local text_width = #text * 4
        local x = (128 - text_width) / 2
        local y = 60
        print(text, x, y, colors.red)
        
        -- instruction to restart
        local restart_text = "click to restart"
        local restart_width = #restart_text * 4
        local restart_x = (128 - restart_width) / 2
        print(restart_text, restart_x, y + 10, colors.white)
        return
    elseif game_state == "countdown" then
        -- draw countdown with animation
        local color = countdown_colors[countdown] or 7
        local scale = countdown_scale
        local num_width = 4 * scale
        local x = 64 - num_width/2
        local y = 64 - num_width/2
        
        print(countdown, x, y, color)
        return -- skip other drawing during countdown
    end
   
    
    -- draw coins
    for c in all(coins) do
        if not c.collected then
            spr(1, c.x-4, c.y-4) -- assuming sprite 1 is a coin (8x8)
        end
    end
    
    -- draw obstacle sprites
    for o in all(obstacles) do
        spr(2, o.x-4, o.y-4) -- sprite 2 is the obstacle sprite (8x8)
    end
    
    -- draw particles
    for p in all(particles) do
        circfill(p.x, p.y, r, p.col)
    end
    
    -- draw cursor
    local cursor_color = colors.red
    local inner_color = colors.white
    
    if mouse_btn == 1 then
        cursor_color = left_cooldown > 0 and colors.red or colors.orange
    elseif mouse_btn == 2 then
        cursor_color = right_cooldown > 0 and colors.red or colors.yellow
        inner_color = is_grabbing and colors.blue or colors.white
    end
    
    if mouse_btn == 2 and not is_grabbing and right_cooldown <= 0 then
        circ(mx, my, grab_radius, colors.yellow)
    end
    
    circ(mx, my, 6, cursor_color)
    circfill(mx, my, 2, inner_color)
    
    -- ui with cooldown and usage display in seconds
    local left_seconds = flr((left_cooldown + 59) / 60) -- round up to show remaining seconds
    local right_seconds = flr((right_cooldown + 59) / 60) -- round up to show remaining seconds
    local left_usage_seconds = flr((max_usage_duration - left_usage_timer) / 60) -- remaining usage time
    local right_usage_seconds = flr((max_usage_duration - right_usage_timer) / 60) -- remaining usage time
    
    print("coins: "..#coins, 2, 2, colors.white)
    print("obstacle: "..(#obstacles > 0 and "active" or "none"), 2, 34, #obstacles > 0 and colors.red or colors.white) -- show obstacle status
    
    -- show cooldown or usage time
    if left_cooldown > 0 then
        print("left: cooldown ("..left_seconds.."s)", 2, 10, colors.red)
    elseif mouse_btn == 1 and left_usage_timer > 0 then
        print("left: using ("..left_usage_seconds.."s)", 2, 10, colors.orange)
    else
        print("left: attract (ready)", 2, 10, colors.white)
    end
    
    if right_cooldown > 0 then
        print("right: cooldown ("..right_seconds.."s)", 2, 18, colors.red)
    elseif mouse_btn == 2 and right_usage_timer > 0 then
        print("right: using ("..right_usage_seconds.."s)", 2, 18, colors.yellow)
    else
        print("right: grab (ready)", 2, 18, colors.white)
    end
    
    if is_grabbing then print("grabbed", 2, 26, colors.blue) end
end
