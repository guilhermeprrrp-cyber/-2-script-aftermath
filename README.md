do
	local gundata = game:GetService("ReplicatedStorage").GunSystemAssets.GunData
    local sv_config = game:GetService("ReplicatedStorage").CustomCharacterConfigs.Configuration.Server
    _IsDescendantOf = game.IsDescendantOf
    _FindFirstChild = game.FindFirstChild
    _FindFirstChildOfClass = game.FindFirstChildOfClass
    _Raycast = workspace.Raycast
    _IsKeyDown = uis.IsKeyDown
    _WorldToViewportPoint = cam.WorldToViewportPoint
    _Vector3zeromin = Vector3.zero.Min
    _Vector2zeromin = Vector2.zero.Min
    _Vector3zeromax = Vector3.zero.Max
    _Vector2zeromax = Vector2.zero.Max
    _IsA = game.IsA

    local entitylist
    for _, gc in getgc(true) do
        if type(gc) == "table" then
            local gpfwc = rawget(gc, "GetPlayerFromWorldCharacter")
            if gpfwc and type(gpfwc) == "function" then
                local upvs = getupvalues(gpfwc)
                local GetCharacters = upvs[2].GetCharacters
                entitylist = getupvalues(GetCharacters)[1]
                break
            end
        end
    end

    local function get_closest_target(fov_size, aimpart, team_check)
        local ermm_part, plr_instance, collider
        local maximum_distance = fov_size
        local mousepos = uis:GetMouseLocation()
        for userid, v in entitylist do
            local player = v.Player
            if not (player and player ~= lp) then continue end
            
            if team_check and type(player) ~= "table" and teamcheck(player) then continue end
            
            local root = v.RootPart
            local worldmodel = v.WorldModel
            local character = v.Character
            if not (root and worldmodel and character) then continue end
            if type(player) == "table" then
                player = {
                    ["Name"] = character.Name .. " (bot)",
                    ["DisplayName"] = character.Name .. " (bot)"
                }
            end
            local part = worldmodel:FindFirstChild(aimpart)
            if not part then continue end
            local position, onscreen = cam:WorldToViewportPoint(part.Position)
            local distance = (Vector2.new(position.X, position.Y) - mousepos).Magnitude
            if onscreen and distance <= maximum_distance then
                plr_instance = player
                ermm_part = part
                collider = root
                maximum_distance = distance
            end
        end
        return ermm_part, plr_instance, collider
    end

    local function predict_target(origin, position, velocity, projectile_speed, projectile_drop)
        local distance = (origin - position).Magnitude
        local time_to_hit = (distance / projectile_speed)
        local predicted_position = position + (velocity * time_to_hit)
        local delta = (predicted_position - position).Magnitude
        time_to_hit = time_to_hit + (delta / projectile_speed)
        return predicted_position + Vector3.yAxis * (projectile_drop * time_to_hit ^ 2)
    end

    local function get_current_gun(plr)
        if not plr then return "Fists" end
        local currentobj = plr:FindFirstChild("CurrentSelectedObject") and plr.CurrentSelectedObject.Value
        local gunname = currentobj and currentobj.Value
        return gunname and gunname.Name or "Fists"
    end

    local function full_prediction(target_position, target_collider)
        if not (target_position) then return end
        local currentgun = get_current_gun(lp)
        local gun = gundata:FindFirstChild(currentgun)
        local stats = gun and gun:FindFirstChild("Stats")
        local bullet_settings = stats and stats:FindFirstChild("BulletSettings")
        if bullet_settings then
            local bullet_speed = bullet_settings:FindFirstChild("BulletSpeed")
            local bullet_drop  = bullet_settings:FindFirstChild("BulletGravity")
            local proj_speed   = bullet_speed and bullet_speed.Value or sv_config.sv_default_bullet_speed.Value or 1500
            local proj_drop    = bullet_drop and bullet_drop.Value and sv_config.sv_default_bullet_gravity.Value or 0
            local campos       = cam.CFrame.Position
            local predicted    = predict_target(
                campos,
                target_position,
                target_collider and target_collider.AssemblyLinearVelocity or Vector3.zero,
                proj_speed, proj_drop
            )
            return predicted
        end
        return target_position
    end

    local old
    old = hookfunction(buffer.create, newcclosure(function(size, ...)
        if size ~= 300 then
            return old(size, ...)
        end
        if not debug.traceback():find("GunController") then
            return old(size, ...)
        end
        local stack = debug.getstack(3, 1)
        if type(stack) ~= "table" then
            return old(size, ...)
        end
        if type(stack[3]) == "table" and stack[3].Resimulation ~= nil then
            return old(size, ...)
        end
        local cam = workspace.CurrentCamera
        local pitch, yaw, roll = cam.CFrame:ToEulerAnglesYXZ()

        local pred
        local part, player, collider = get_closest_target(
            fovsize,
            "Head",
            true  
        )

        if part then
            pred = full_prediction(part.Position, collider)
        end

        local ld

        if pred and silentaimenabled and aimbotEnabled then
            ld = CFrame.lookAt(
                cam.CFrame.Position,
                pred
            )
        else
            ld = cam.CFrame.LookVector
        end

        local spread = Vector3.zero

        if nospreadenabled then
            local rng = Random.new(stack[43] + 1)
            spread = Vector3.new(
                rng:NextNumber() - rng:NextNumber(),
                rng:NextNumber() - rng:NextNumber(),
                rng:NextNumber() - rng:NextNumber()
            ) / stack[18]
        end

        if typeof(ld) == "Vector3" then
            ld = (ld - spread).Unit
        else
            ld = (ld.LookVector - spread).Unit
        end

        local cf = CFrame.lookAt(Vector3.zero, ld)
        local pitch2, yaw2, roll2 = cf:ToEulerAnglesYXZ()
        local dir = cf.LookVector

        local r00, r01, r02, r10, r11, r12, r20, r21, r22 = cf:GetComponents()

        stack[27] = cf
        stack[28] = dir
        stack[29] = dir
        stack[31] = pitch2
        stack[32] = yaw2
        stack[33] = CFrame.new(0, 0, 0, r00, r01, r02, r10, r11, r12, r20, r21, r22)
        stack[34] = CFrame.new(0, 0, 0, r00, r01, r02, r10, r11, r12, r20, r21, r22)
        stack[39] = CFrame.new(0, 0, 0, r00, r01, r02, r10, r11, r12, r20, r21, r22)
        stack[40] = dir
        stack[41] = dir

        return old(size, ...)
    end))
end

norecoiltoggle = AimbotSection:toggle({name = "No Recoil", def = false, callback = function(state)
    norecoilenabled = state
    
    for _, obj in getgc(true) do
        if typeof(obj) == "table" and rawget(obj, "_positionVelocity") and type(rawget(obj, "_positionVelocity")) == "function" then
            original = clonefunction(rawget(obj, "_positionVelocity"))
            rawset(obj, "_positionVelocity", function(self, now)
                if norecoilenabled then
                    return Vector3.zero, Vector3.zero
                else
                    return original(self, now)
                end
            end)
        end
    end
end})

nobobtables = {}
nobobtoggle = AimbotSection:toggle({name = "No Gun Bob", def = false, callback = function(state)
    nobobenabled = state
    
    for _, tbl in getgc(true) do
        if type(tbl) == 'table' and rawget(tbl, 'BobSpeed') then
            if state then
                nobobtables[tbl] = {
                    BobSpeed = tbl.BobSpeed,
                    BobAmplitudeHorizontal = tbl.BobAmplitudeHorizontal,
                    BobAmplitudeVertical = tbl.BobAmplitudeVertical,
                    MovementOffset = tbl.MovementOffset,
                    CrouchOffset = tbl.CrouchOffset,
                    TransitionRate = tbl.TransitionRate,
                    TransitionRateCrouch = tbl.TransitionRateCrouch,
                    BobPower = tbl.BobPower
                }
                
                tbl.BobSpeed = 0
                tbl.BobAmplitudeHorizontal = 0
                tbl.BobAmplitudeVertical = 0
                tbl.MovementOffset = Vector3.new()
                tbl.CrouchOffset = Vector3.new()
                tbl.TransitionRate = 0
                tbl.TransitionRateCrouch = 0
                tbl.BobPower = 0
            else
                saved = nobobtables[tbl]
                if saved then
                    tbl.BobSpeed = saved.BobSpeed
                    tbl.BobAmplitudeHorizontal = saved.BobAmplitudeHorizontal
                    tbl.BobAmplitudeVertical = saved.BobAmplitudeVertical
                    tbl.MovementOffset = saved.MovementOffset
                    tbl.CrouchOffset = saved.CrouchOffset
                    tbl.TransitionRate = saved.TransitionRate
                    tbl.TransitionRateCrouch = saved.TransitionRateCrouch
                    tbl.BobPower = saved.BobPower
                end
            end
        end
    end
end})

local httpservice = game:GetService("HttpService")
local new_decode = function(old_decode, self, ...)
    if checkcaller() or self ~= httpservice then return old_decode(self, ...) end
    if tostring(getcallingscript()) == "GunController" then
        local data = old_decode(self, ...)
        if not (type(data["RPM"]) == "table" and type(data["Auto"]) == "table") then
            return data
        end
        if aimbot_forceauto then
            data.Auto.Value = true
        end
        if aimbot_rapidfire then
            data.RPM.Value *= aimbot_rapidfire_multiplier
        end
        return data
    end
    return old_decode(self, ...)
end

local old_decode; old_decode = hookfunction(httpservice.JSONDecode, newcclosure(LPH_NO_VIRTUALIZE(function(...)
    return new_decode(old_decode, ...)
end)))

local gunfire, bullethole = aftermath.remotes.gunfire, aftermath.remotes.bullethole
local known_indexes = aftermath.packet_manipulation.known_indexes

local new_fireserver = LPH_JIT_MAX(function(old_fireserver, self, ...)
    if self == gunfire then
        local args = {...}
        if typeof(args[1]) ~= "buffer" then
            return old_fireserver(self, ...)
        end

        local serialized = aftermath.packet_manipulation.serialize(args[1], "gunfire")
        
        local known_gunfire = known_indexes["gunfire"]
        
        if aimbot_forceauto or aimbot_rapidfire then
            local currentgun = get_current_gun(LocalPlayer)
            local gun = _FindFirstChild(aftermath.gundata, currentgun)
            local stats = gun and _FindFirstChild(gun, "Stats")
            local rpm, auto = stats and _FindFirstChild(stats, "RPM"), stats and _FindFirstChild(stats, "Auto")
            if not (rpm and auto) then
                return old_fireserver(self, ...)
            end

            if aimbot_forceauto and not auto.Value then
                serialized[known_gunfire["statchecksum"]].data -= 11
            end
            if aimbot_rapidfire then
                local original = rpm.Value
                serialized[known_gunfire["rpm1"]].data = original
                serialized[known_gunfire["rpm2"]].data = original
            end
        end
        
        serialized[known_gunfire["time"]].data = 0/0

        args[1] = aftermath.packet_manipulation.encode(serialized)
        return old_fireserver(self, unpack(args))
    end

    return old_fireserver(self, ...)
end)

local old_fireserver; old_fireserver = hookfunction(gunfire.FireServer, newcclosure(LPH_NO_VIRTUALIZE(function(...)
    return new_fireserver(old_fireserver, ...)
end)))
