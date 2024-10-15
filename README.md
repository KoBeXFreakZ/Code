AddCSLuaFile("cl_init.lua")
AddCSLuaFile("shared.lua")
include('shared.lua')

ENT.Base				= "base_nextbot"
ENT.Spawnable			= true
ENT.AdminOnly			= false
ENT.PrintName		= "Richard Boderman"
ENT.Author			= "Fay"
ENT.Information		= ""

ENT.Model = "models/fay/boderman/boderman.mdl" 
--models/Characters/Hostage_01.mdl

ENT.MeleeSounds = {"scientist/holdstill.wav","scientist/greetings.wav"}
ENT.RunSounds = {"scientist/noidea.wav"}
ENT.PainSounds = {"scientist/noplease.wav","scientist/sci_pain9.wav","scientist/scream11.wav","scientist/c1a3_sci_silo2a.wav"}
ENT.DeathSounds = {"scientist/scream24.wav","scientist/scream21.wav","scientist/scream20.wav"}

ENT.ConstSound = {"gamer/fnaf_amb.mp3","gamer/fnaf_amb2.mp3","gamer/valk_amb.mp3"}
ENT.ConstSoundLength = {604,10,300}

function ENT:Initialize()
	self:SetCollisionBounds( Vector(0,0,0), Vector(0,0,64) ) -- make sure the zombie doesn't get stuck
	self:SetCollisionGroup(COLLISION_GROUP_WORLD)
	
	self.LoseTargetDist	= 9999999999
	self.SearchRadius 	= 99999999
	
	self.playingPainSound = false
	
	self:SetModel(self.Model)
	
	self:SetHealth(tonumber(GetConVar("boderman_start_health"):GetString()))
	
	self.loco:SetJumpHeight(300)
	
	self.lastTime = CurTime() + 30
	
	self:SetSequence( "walk" )
	self:StartActivity(ACT_IDLE)
	
	self.MovePath = Path("Follow")
	
	self.loopSNDConst = 0
	
	self.pickedSND = math.random(1,#self.ConstSound)
end

local function BDPostInitEntity()
	CreateConVar("boderman_target_npcs", "0", FCVAR_NONE, "(1 or 0) Whether Boderman should attack NPCs or not")
	CreateConVar("boderman_target_player_specific", "", FCVAR_NONE, "(SteamID64) (Multiplayer Only) If set, forces Boderman to attack player with specified SteamID64, ignoring any other flags that would prevent it from attacking this specific player")
	CreateConVar("boderman_start_health", "10000", FCVAR_NONE, "(number) The starting health that Boderman will have when spawned")
	CreateConVar("boderman_move_speed", "900", FCVAR_NONE, "(number) The speed at which Boderman will move at while pursuing enemies")
	CreateConVar("boderman_target_players", "1", FCVAR_NONE, "(1 or 0) Whether Boderman should attack players or not")
	CreateConVar("boderman_teleport_toggle", "1", FCVAR_NONE, "(1 or 0) Whether Boderman should teleport if he is too far from an enemy")
	CreateConVar("boderman_teleport_timer", "10", FCVAR_NONE, "(number) Number of seconds between Boderman teleport chances")
	CreateConVar("boderman_teleport_distance", "2000000", FCVAR_NONE, "(number) How far Boderman must be from the player (in units squared) for a teleport chance to occur")
	CreateConVar("boderman_starting_timer", "5", FCVAR_NONE, "(number) How long Boderman will wait after being spawned to start going towards enemies")
	CreateConVar("boderman_teleport_chance", "30", FCVAR_NONE, "(0 to 100) The percentage that Boderman has of teleporting if he is far away from all enemies")
	CreateConVar("boderman_stuckclip_toggle", "1", FCVAR_NONE, "(0 or 1) Whether Boderman should easily clip through doors and thin walls if stuck")
	CreateConVar("boderman_wander_toggle", "1", FCVAR_NONE, "(0 or 1) Whether Boderman should wander around randomly when no enemies are present")
	CreateConVar("boderman_play_amb", "1", FCVAR_NONE, "(0 or 1) Whether Boderman should play ambient spooky music that gets louder as he gets closer to you")
end

hook.Add( "InitPostEntity", "Boderman_IPEHook", BDPostInitEntity)

ENT.LastStuck = 0
ENT.StuckTries = 0

function ENT:OnStuck()
	-- Jump forward a bit on the path.
	self.LastStuck = CurTime()
	
	if GetConVar("boderman_stuckclip_toggle"):GetString() == "1" then
		self:SetEnemy(self:GetNearestTarget())

		local newCursor = self.MovePath:GetCursorPosition()
			+ 40 * math.pow(2, self.StuckTries)
		self:SetPos(self.MovePath:GetPositionOnPath(newCursor))
		self.StuckTries = self.StuckTries + 1
	end

	-- Hope that we're not stuck any more.
	self.loco:ClearStuck()
end

function ENT:UnstickFromCeiling()
	if self:IsOnGround() then return end

	-- NextBots LOVE to get stuck. Stuck in the morning. Stuck in the evening.
	-- Stuck in the ceiling. Stuck on each other. The stuck never ends.
	local myPos = self:GetPos()
	local myHullMin, myHullMax = self:GetCollisionBounds()
	local myHull = myHullMax - myHullMin
	local myHullTop = myPos + vector_up * myHull.z
	trace.start = myPos
	trace.endpos = myHullTop
	trace.filter = self
	local upTrace = util.TraceLine(trace, self)

	if upTrace.Hit and upTrace.HitNormal ~= vector_origin
		and upTrace.Fraction > 0.5
	then
		local unstuckPos = myPos
			+ upTrace.HitNormal * (myHull.z * (1 - upTrace.Fraction))
		self:SetPos(unstuckPos)
	end
end

----------------------------------------------------
-- ENT:Get/SetEnemy()
-- Simple functions used in keeping our enemy saved
----------------------------------------------------
function ENT:SetEnemy(ent)
	self.Enemy = ent
end
function ENT:GetEnemy()
	return self.Enemy
end

----------------------------------------------------
-- ENT:HaveEnemy()
-- Returns true if we have a enemy
----------------------------------------------------
function ENT:HaveEnemy()
	self:SetEnemy(self:GetNearestTarget())
	return self:GetNearestTarget()
end

function ENT:OnInjured( info )
	if not self.playingPainSound && self:Health() - info:GetDamage() > 0 then
		self.playingPainSound = true
		self:EmitSound( self.PainSounds[math.random(1,#self.PainSounds)], 100, 90 )
		timer.Simple( 2.5, function() self.playingPainSound = false end )
	end
end

local HIGH_JUMP_HEIGHT = 500
function ENT:AttemptJumpAtTarget()
	-- No double-jumping.
	if not self:IsOnGround() then return end

	local targetPos = self:GetEnemy():GetPos()
	local xyDistSqr = (targetPos - self:GetPos()):Length2DSqr()
	local zDifference = targetPos.z - self:GetPos().z
	local maxAttackDistance = 100
	if xyDistSqr <= math.pow(maxAttackDistance + 200, 2)
		and zDifference >= maxAttackDistance
	then
		--TODO: Set up jump so target lands on parabola.
		local jumpHeight = zDifference + 50
		self.loco:SetJumpHeight(jumpHeight)
		self.loco:Jump()
		self:StartActivity( ACT_RUN )
		self.loco:SetJumpHeight(300)
	end
end

function ENT:OnKilled( dmginfo )

	self.loco:SetDesiredSpeed( 0 )		-- Set the speed that we will be moving at. Don't worry, the animation will speed up/slow down to match
	self.loco:SetAcceleration(0)
	
	self:EmitSound( self.DeathSounds[math.random(1,#self.DeathSounds)], 100, 90 )
	
	hook.Call( "OnNPCKilled", GAMEMODE, self, dmginfo:GetAttacker(), dmginfo:GetInflictor() )
	
	self:StartActivity(ACT_DIESIMPLE)

	--self:BecomeRagdoll( dmginfo )

end

----------------------------------------------------
-- ENT:FindEnemy()
-- Returns true and sets our enemy if we find one
----------------------------------------------------

function ENT:PercentageFrozen()
	return 0
end

----------------------------------------------------
-- ENT:RunBehaviour()
-- This is where the meat of our AI is
----------------------------------------------------
function ENT:RunBehaviour()
	-- This function is called when the entity is first spawned. It acts as a giant loop that will run as long as the NPC exists
	if GetConVar("ai_disabled"):GetBool() == true then
		repeat coroutine.wait(1) until GetConVar("ai_disabled"):GetBool() == false
	end
	
	self:EmitSound( self.RunSounds[math.random(1,#self.RunSounds)], 100, 90 )
	coroutine.wait(tonumber(GetConVar("boderman_starting_timer"):GetString()))
	--self.loopSNDConst = self:StartLoopingSound(self.ConstSound[math.random(1,#self.ConstSound)])
	
	local bdmn = self
	timer.Create( "boderman_sndloop_"..tostring(bdmn:GetCreationID()), 1, 0, function()
		bdmn.loopSNDConst = bdmn.loopSNDConst - 1
		if bdmn.loopSNDConst <= 0 and tonumber(GetConVar("boderman_play_amb"):GetString()) != 0 then
			bdmn:EmitSound(bdmn.ConstSound[bdmn.pickedSND])
			bdmn.loopSNDConst = bdmn.ConstSoundLength[bdmn.pickedSND]
		end
	end )
	
	while ( self:Health() > 0 ) do
		if GetConVar("ai_disabled"):GetBool() == false then
			-- Lets use the above mentioned functions to see if we have/can find a enemy
			if ( self:HaveEnemy() ) then
				-- Now that we have an enemy, the code in this block will run
				self.loco:FaceTowards(self:GetEnemy():GetPos())	-- Face our enemy
				self:StartActivity( ACT_RUN )			-- Set the animation
				self.loco:SetDesiredSpeed( tonumber(GetConVar("boderman_move_speed"):GetString()) )		-- Set the speed that we will be moving at. Don't worry, the animation will speed up/slow down to match
				self.loco:SetAcceleration(10000*(tonumber(GetConVar("boderman_move_speed"):GetString())/900))			-- We are going to run at the enemy quickly, so we want to accelerate really fast
				self:ChaseEnemy( ) 						-- The new function like MoveToPos.
				if(IsValid(self:GetEnemy()) && self:Health() > 0) then
					if(self:GetEnemy():GetPos():DistToSqr( self:GetPos() ) <= 10000 && self:GetEnemy():Health() > 0) then
						self:GetEnemy():PrecacheGibs()
						timer.Simple( 0.025, function() self:GetEnemy():SetVelocity( self:GetForward()*75000 + self:GetUp()*5000 ) self:GetEnemy():Kill() self:EmitSound( self.MeleeSounds[math.random(1,#self.MeleeSounds)], 100, 90 ) end )
						self:PlaySequenceAndWait( "attack1" )		-- Lets make a pose to show we found a enemy
					end
				end
				if(self:Health() > 0)then
					self.loco:SetAcceleration(500*(tonumber(GetConVar("boderman_move_speed"):GetString())/900))			-- Set this back to its default since we are done chasing the enemy
					self:StartActivity( ACT_IDLE )			--We are done so go back to idle
				end
				-- Now once the above function is finished doing what it needs to do, the code will loop back to the start
				-- unless you put stuff after the if statement. Then that will be run before it loops
			elseif GetConVar("boderman_wander_toggle"):GetString() == "1" then
				--LOOK INTO THIS, MAYBE ALLOW MOVING AROUND BUT CANCEL MOVEMENT AFTER A FEW SECONDS?
				-- Since we can't find an enemy, lets wander
				-- Its the same code used in Garry's test bot
				self:SetSequence( "walk" )			-- Walk anmimation
				self.loco:SetDesiredSpeed( tonumber(GetConVar("boderman_move_speed"):GetString())*(200/900) )		-- Walk speed
				local wanderOp = {
					maxage = 5
				}
				self:MoveToPos( self:GetPos() + Vector( math.Rand( -1, 1 ), math.Rand( -1, 1 ), 0 ) * 600, wanderOp) -- Walk to a random place within about 400 units (yielding)
				self:StartActivity( ACT_IDLE )
			end
			-- At this point in the code the bot has stopped chasing the player or finished walking to a random spot
			-- Using this next function we are going to wait 2 seconds until we go ahead and repeat it 
		end
			
		coroutine.wait(1)
		
	end
	timer.Remove( "boderman_sndloop_"..tostring(self:GetCreationID()))
	for _,soundName in ipairs(self.ConstSound) do
		self:StopSound(soundName)
	end
end	

function ENT:OnRemove()
	timer.Remove( "boderman_sndloop_"..tostring(self:GetCreationID()))
	for _,soundName in ipairs(self.ConstSound) do
		self:StopSound(soundName)
	end
end

local hardWhiteListed = { -- things that mess up when not allowed
    ["worldspawn"] = true, -- constraints with the world
    ["gmod_anchor"] = true -- used in slider constraints with world
}

local function isValidTarget(ent)
	-- Ignore non-existant entities.
	if not IsValid(ent) then return false end

	-- Ignore dead NPCs, other Sanics, and dummy NPCs.
	local class = ent:GetClass()
	if ent:IsPlayer() && ent:IsBot() == false && ent:IsNPC() == false && game.SinglePlayer() == false then 
		return (ent:IsPlayer() and GetConVar("boderman_target_players"):GetString() == "1" and GetConVar("ai_ignoreplayers"):GetBool() == false and GetConVar("boderman_target_player_specific"):GetString() == "" and ent:Health() > 0 and not ent:IsFlagSet( FL_NOTARGET ) 
			or ent:IsNPC() and GetConVar("boderman_target_npcs"):GetString() == "1" and GetConVar("boderman_target_player_specific"):GetString() == "" and ent:Health() > 0 and not ent:IsFlagSet( FL_NOTARGET ) 
			or GetConVar("boderman_target_player_specific"):GetString() == tostring(ent:SteamID64()) and ent:Health() > 0)
	else
		return (ent:IsPlayer() and GetConVar("boderman_target_players"):GetString() == "1" and GetConVar("ai_ignoreplayers"):GetBool() == false and GetConVar("boderman_target_player_specific"):GetString() == "" and ent:Health() > 0 and not ent:IsFlagSet( FL_NOTARGET ) 
			or ent:IsNPC() and GetConVar("boderman_target_npcs"):GetString() == "1" and GetConVar("boderman_target_player_specific"):GetString() == "" and ent:Health() > 0 and not ent:IsFlagSet( FL_NOTARGET ) )
	end
end

function ENT:AttackNearbyTargets(radius)
	if GetConVar("ai_disabled"):GetBool() == false then
		local attackForce = 800
		local hitSource = self:LocalToWorld(self:OBBCenter())
		local nearEntities = ents.FindInSphere(hitSource, radius)
		local hit = false
		for _, ent in pairs(nearEntities) do
			if IsValid(ent) then
				if isValidTarget(ent) then
					local health = ent:Health()

					if ent:IsPlayer() and IsValid(ent:GetVehicle()) then
						-- Hiding in a vehicle, eh?
						local vehicle = ent:GetVehicle()

						local vehiclePos = vehicle:LocalToWorld(vehicle:OBBCenter())
						local hitDirection = (vehiclePos - hitSource):GetNormal()

						-- Give it a good whack.
						local phys = vehicle:GetPhysicsObject()
						if IsValid(phys) then
							phys:Wake()
							local hitOffset = vehicle:NearestPoint(hitSource)
							phys:ApplyForceOffset(hitDirection
								* (attackForce * phys:GetMass()),
								hitOffset)
						end
						vehicle:TakeDamage(math.max(1e8, ent:Health()), self, self)

						-- Oh, and make a nice SMASH noise.
						vehicle:EmitSound(string.format(
							"physics/metal/metal_sheet_impact_hard%d.wav",
							math.random(6, 8)), 350, 120)
					else
						ent:EmitSound(string.format(
							"physics/body/body_medium_impact_hard%d.wav",
							math.random(1, 6)), 350, 120)
					end

					local hitDirection = (ent:GetPos() - hitSource):GetNormal()
					-- Give the player a good whack. Sanic means business.
					-- This is for those with god mode enabled.
					ent:SetVelocity(hitDirection * attackForce + vector_up * 500)

					local dmgInfo = DamageInfo()
					dmgInfo:SetAttacker(self)
					dmgInfo:SetInflictor(self)
					dmgInfo:SetDamage(1e8)
					dmgInfo:SetDamagePosition(self:GetPos())
					dmgInfo:SetDamageForce((hitDirection * attackForce
						+ vector_up * 500) * 100)
					ent:TakeDamageInfo(dmgInfo)

					local newHealth = ent:Health()

					-- Hits only count if we dealt some damage.
					hit = (hit or (newHealth < health))
					self:PlaySequenceAndWait( "attack1" )		-- Lets make a pose to show we found a enemy
					self:StartActivity( ACT_RUN )
				elseif ent:GetMoveType() == MOVETYPE_VPHYSICS then
					if ent:IsVehicle() and IsValid(ent:GetDriver()) then continue end
					if ent.CPPIGetOwner then
						if not IsValid(ent:CPPIGetOwner()) then
							continue
						end
					end

					-- Knock away any props put in our path.
					local entPos = ent:LocalToWorld(ent:OBBCenter())
					local hitDirection = (entPos - hitSource):GetNormal()
					local hitOffset = ent:NearestPoint(hitSource)

					-- Remove anything tying the entity down.
					-- We're crashing through here!
					constraint.RemoveAll(ent)

					-- Get the object's mass.
					local phys = ent:GetPhysicsObject()
					local mass = 0
					local material = "Default"
					if IsValid(phys) then
						mass = phys:GetMass()
						material = phys:GetMaterial()
					end

					-- Don't make a noise if the object is too light.
					-- It's probably a gib.
					if mass >= 5 then
						ent:EmitSound(material .. ".ImpactHard", 350, 120)
					end

					-- Unfreeze all bones, and give the object a good whack.
					for id = 0, ent:GetPhysicsObjectCount() - 1 do
						local phys = ent:GetPhysicsObjectNum(id)
						if IsValid(phys) then
							phys:EnableMotion(true)
							phys:ApplyForceOffset(hitDirection * (attackForce * mass),
								hitOffset)
						end
					end

					-- Deal some solid damage, too.
					ent:TakeDamage(25, self, self)
					
					self:PlaySequenceAndWait( "attack1" )		-- Lets make a pose to show we found a enemy
					self:StartActivity( ACT_RUN )
				end
			end
		end
	end

	return hit
end

function ENT:GetNearestTarget()
	-- Only target entities within the acquire distance.
	if (GetConVar("ai_disabled"):GetBool() == false) then
		local maxAcquireDist = 99999999
		local maxAcquireDistSqr = maxAcquireDist * maxAcquireDist
		local myPos = self:GetPos()
		local acquirableEntities = ents.FindInSphere(myPos, maxAcquireDist)
		local distToSqr = myPos.DistToSqr
		local getPos = self.GetPos
		local target = nil
		local getClass = self.GetClass

		for _, ent in pairs(acquirableEntities) do
			-- Ignore invalid targets, of course.
			if not isValidTarget(ent) then continue end

			-- Spawn protection! Ignore players within 200 units of a spawn point
			-- if `npc_sanic_spawn_protect' = 1.
			--TODO: Only for the first few seconds?

			-- Find the nearest target to chase.
			local distSqr = distToSqr(getPos(ent), myPos)
			if distSqr < maxAcquireDistSqr then
				target = ent
				maxAcquireDistSqr = distSqr
			end
		end

		return target
	else
		return nil
	end
end

----------------------------------------------------
-- ENT:ChaseEnemy()
-- Works similarly to Garry's MoveToPos function
--  except it will constantly follow the
--  position of the enemy until there no longer
--  is one.
----------------------------------------------------

local function isEmpty(vector, ignore)
    ignore = ignore or {}

    local point = util.PointContents(vector)
    local a = point ~= CONTENTS_SOLID
        and point ~= CONTENTS_MOVEABLE
        and point ~= CONTENTS_LADDER
        and point ~= CONTENTS_PLAYERCLIP
        and point ~= CONTENTS_MONSTERCLIP
    if not a then return false end

    local b = true

    for _, v in ipairs(ents.FindInSphere(vector, 35)) do
        if (v:IsNPC() or v:IsPlayer() or v:GetClass() == "prop_physics" or v.NotEmptyPos) and not table.HasValue(ignore, v) then
            b = false
            break
        end
    end

    return a and b
end

local function findEmptyPos(pos, ignore, distance, step, area)
    if isEmpty(pos, ignore) and isEmpty(pos + area, ignore) then
        return pos
    end

    for j = step, distance, step do
        for i = -1, 1, 2 do -- alternate in direction
            local k = j * i

            -- Look North/South
            if isEmpty(pos + Vector(k, 0, 0), ignore) and isEmpty(pos + Vector(k, 0, 0) + area, ignore) then
                return pos + Vector(k, 0, 0)
            end

            -- Look East/West
            if isEmpty(pos + Vector(0, k, 0), ignore) and isEmpty(pos + Vector(0, k, 0) + area, ignore) then
                return pos + Vector(0, k, 0)
            end

            -- Look Up/Down
            if isEmpty(pos + Vector(0, 0, k), ignore) and isEmpty(pos + Vector(0, 0, k) + area, ignore) then
                return pos + Vector(0, 0, k)
            end
        end
    end

    return pos
end

function ENT:ChaseEnemy( options )

	local options = options or {}
	self.MovePath = Path( "Follow" )
	self.MovePath:SetMinLookAheadDistance( options.lookahead or 300 )
	self.MovePath:SetGoalTolerance( options.tolerance or 10 )
	self.MovePath:Compute( self, self:GetEnemy():GetPos() )		-- Compute the path towards the enemies position

	if ( !self.MovePath:IsValid() ) then return "failed" end

	while ( self.MovePath:IsValid() and self:HaveEnemy() && self:Health() > 0) && GetConVar("ai_disabled"):GetBool() == false do
		self:SetEnemy(self:GetNearestTarget())
		if ( self.MovePath:GetAge() > 0.1 ) then					-- Since we are following the player we have to constantly remake the path
			self.MovePath:Compute(self, self:GetEnemy():GetPos())-- Compute the path towards the enemy's position again
		end
		self.MovePath:Update( self )								-- This function moves the bot along the path
		self:AttemptJumpAtTarget()
		self:AttackNearbyTargets(100)
		local en = self:GetEnemy()
		xpcall( function() 
			if en ~= nil then
				if(en:GetPos():DistToSqr( self:GetPos() ) <= 10000 && en:Health() > 0) then
					en:PrecacheGibs()
					timer.Simple( 0.025, function() if self:GetEnemy() ~= nil then en:SetVelocity( self:GetForward()*75000 + self:GetUp()*5000 ) en:Kill() self:EmitSound( self.MeleeSounds[math.random(1,#self.MeleeSounds)], 100, 90 ) end end )
					self:PlaySequenceAndWait( "attack1" )		-- Lets make a pose to show we found a enemy
					return "ok"
				end
				if(en:GetPos():DistToSqr( self:GetPos() ) >= tonumber(GetConVar("boderman_teleport_distance"):GetString()) and GetConVar("boderman_teleport_toggle"):GetString() == "1") then
					if(CurTime() - self.lastTime >= tonumber(GetConVar("boderman_teleport_timer"):GetString())) then
						self.lastTime = CurTime()
						if(math.Rand(1,10) <= tonumber(GetConVar("boderman_teleport_chance"):GetString())/10) then
							self:SetPos(findEmptyPos(en:GetPos(), {}, 3000, 1500, Vector(4, 4, 64)))
						end
					end
				end
			
				if(en:GetPos():DistToSqr( self:GetPos() ) <= 250000) then
					self:SetSequence( "runshort" )
				else
					self:SetSequence( "runlong" )
				end
				
				if ( options.draw ) then self.MovePath:Draw() end
				-- If we're stuck, then call the HandleStuck function and abandon
				if ( self.loco:IsStuck() ) then
					self:HandleStuck()
					return "stuck"
				end
			end
		end, function(err)  end)

		coroutine.yield()

	end

	return "ok"

end

local function resetBDVars( player, command, arguments )
	--set convars back to default here
	GetConVar("boderman_target_npcs"):SetString("0")
	GetConVar("boderman_target_player_specific"):SetString("")
	GetConVar("boderman_start_health"):SetString("10000")
	GetConVar("boderman_move_speed"):SetString("900")
	GetConVar("boderman_target_players"):SetString("1")
	GetConVar("boderman_teleport_toggle"):SetString("1")
	GetConVar("boderman_teleport_timer"):SetString("10")
	GetConVar("boderman_teleport_distance"):SetString("2000000")
	GetConVar("boderman_starting_timer"):SetString("5")
	GetConVar("boderman_teleport_chance"):SetString("30")
	GetConVar("boderman_stuckclip_toggle"):SetString("1")
	GetConVar("boderman_wander_toggle"):SetString("1")
	return true
end
 
concommand.Add( "boderman_reset_defaults", resetBDVars, nil, "Resets all Boderman related convars back to their default values" )
