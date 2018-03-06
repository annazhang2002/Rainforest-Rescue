# Rainforest-Rescue
pico-8 cartridge // http://www.pico-8.com
version 8
__lua__
-- vector library
function v_init(x,y)
	local v1 = {}
	v1.x = x
	v1.y = y
	return v1
end

function v_add(v1,v2)
	return v_init(v1.x + v2.x, v1.y + v2.y)
end

function v_sub(v1,v2)
	return v_init(v1.x - v2.x, v1.y - v2.y)
end

function v_div(v1, s)
	return v_init(v1.x / s, v1.y / s)
end

function v_mul(v1, s)
	return v_init(v1.x * s, v1.y * s)
end

function v_mag(v1)
	return sqrt((v1.x * v1.x) + (v1.y * v1.y))
end

function v_norm(v1)
	return v_div(v1, v_mag(v1))
end

-- debug library
deb_mode_on = false
deb_states = {"normal","step", "slowmo"}
deb_curt_state = 1
deb_frame_timer = 0

deb_log_buffer = {}
deb_text_buffer = {}
deb_rect_buffer = {}

function deb_start()
	deb_frame_timer = 0
	deb_mode_on = not deb_mode_on
end

function deb_log(msg)
	add(deb_log_buffer, msg)
end

function deb_text(msg, x, y)
	local text_entry = {}
	text_entry.msg = msg
	text_entry.x = x
	text_entry.y = y
	add(deb_text_buffer, text_entry)
end

function deb_rect(x1, y1, x2, y2)
	local rect_entry = {}
	rect_entry.x1 = x1
	rect_entry.y1 = y1
	rect_entry.x2 = x2
	rect_entry.y2 = y2
	add(deb_rect_buffer, rect_entry)
end

function deb_clear_buffers()
	deb_text_buffer = {}
	deb_rect_buffer = {}
	deb_log_buffer  = {}
end

-- physics library
gravity  = v_init(0, 0.1)
air_fric = 0.05

function phy_aabb_collide(pos_a, a, pos_b, b)
	if(pos_a.x + a.max.x < pos_b.x + b.min.x or
	   pos_b.x + b.max.x < pos_a.x + a.min.x) then
		return false
	end

	if(pos_a.y + a.max.y < pos_b.y + b.min.y or
	   pos_b.y + b.max.y < pos_a.y + a.min.y) then
		return false
	end

	return true
end

function phy_aabb_init(x,y,w,h)
	local aabb = {}
	aabb.w = w
	aabb.h = h
	aabb.min = v_init(x,y)
	aabb.max = v_init(x+w,y+h)
	return aabb
end

function apply_force(obj, f)
	obj.acc.x += f.x/obj.mass
	obj.acc.y += f.y/obj.mass
end

function update_physics(obj)

	-- apply gravity
	local f_gravity = v_mul(gravity, obj.mass)
  if (obj.grav==true) then
		apply_force(obj, f_gravity)
  end
	-- apply air resistence
	if(v_mag(obj.vel) > 0) then
		local opp_vel = v_mul(v_norm(obj.vel), -1)
		local vel_mag = v_mag(obj.vel)
		local air_res = v_mul(opp_vel, air_fric)
		apply_force(obj, v_mul(air_res, vel_mag*vel_mag))
	end

	obj.vel = v_add(obj.vel, obj.acc)

	if(abs(obj.vel.x) > 0) then
		phy_check_horz_coll(obj)
	end

	if(abs(obj.vel.y) > 0) then
		phy_check_vert_coll(obj)
	end

	obj.pos = v_add(obj.pos, obj.vel)
	obj.acc = v_init(0, 0)
end

function phy_is_solid(x, y)
	local value = mget(x, y)
	return fget(value, 0)
end

function phy_is_building(x,y)
	local value = mget(x,y)
	return fget(value, 1)
end

function phy_is_fire(x,y)
	local value = mget(x,y)
	return fget(value, 2)
end

function phy_finish_level(x,y)
	local value = mget(x,y)
	return fget(value, 3)
end

function phy_is_co2(x,y)
	local value = mget(x,y)
	return fget(value, 4)
end

function phy_is_macaw(x,y)
	local macaw_x = macaw.pos.x
	local macaw_y = macaw.pos.y
	local macaw = phy_screen_to_map(v_init(macaw_x,macaw_y))
	if x > macaw.x and x < macaw.x +2 then
		if y > macaw.y and y < macaw.y + 2 then
			return true
		end
	end
end

function phy_screen_to_map(pos)
	return v_div(pos, 8)
end

function phy_map_to_screen(pos)
	return v_mul(pos, 8)
end

function phy_check_horz_coll(obj)

	local proj_pos = v_init(obj.pos.x + obj.vel.x, obj.pos.y)

	local side = sgn(obj.vel.x)

	local t_corner = v_init(proj_pos.x, proj_pos.y)
	if(side == 1) then
		t_corner.x += 7
	end

	local b_corner = v_init(proj_pos.x, proj_pos.y + 7)
	if(side == 1) then
		b_corner.x += 7
	end

	-- transform world pos of both corners to map pos
	local t_map_pos = v_div(t_corner,8)
	local b_map_pos = v_div(b_corner,8)

	-- check in the map if the value in both map pos are solid
	if(phy_is_solid(t_map_pos.x, t_map_pos.y) or
	   phy_is_solid(b_map_pos.x, b_map_pos.y)) then
		   obj.vel.x = 0
		   obj.pos.x = (flr(t_map_pos.x) - side) * 8
	end
	if(phy_is_building(t_map_pos.x, t_map_pos.y) or
	   phy_is_building(b_map_pos.x, b_map_pos.y)) then
		   obj.is_playing = false
			 obj.hit_building = true
	end
	if(phy_is_fire(t_map_pos.x, t_map_pos.y) or
	   phy_is_fire(b_map_pos.x, b_map_pos.y)) then
		   obj.is_playing = false
			 obj.hit_fire = true
	end
	if(phy_finish_level(t_map_pos.x, t_map_pos.y) or
	   phy_finish_level(b_map_pos.x, b_map_pos.y)) then
		   obj.is_playing = false
			 obj.finish = true
			 data_just_finished=true
	end
	if(phy_is_co2(t_map_pos.x, t_map_pos.y) or
		 phy_is_co2(b_map_pos.x, b_map_pos.y)) then
			 obj.is_playing = false
			 obj.hit_co2 = true
	end
	if(phy_is_macaw(t_map_pos.x, t_map_pos.y) or
		 phy_is_macaw(b_map_pos.x, b_map_pos.y)) then
			 obj.is_playing = false
			 obj.hit_macaw = true
	end
end

--collision detection

function phy_check_vert_coll(obj)

	-- project our vert position
	local proj_pos = v_init(obj.pos.x, obj.pos.y + obj.vel.y)

	-- find to which vert side i am going
	local side = sgn(obj.vel.y)

	-- find both corners on that side
	local l_corner = v_init(proj_pos.x, proj_pos.y)
	if(side == 1) then
		l_corner.y += 7
	end

	local r_corner = v_init(proj_pos.x + 7, proj_pos.y)
	if(side == 1) then
		r_corner.y += 7
	end

	-- transform world pos of both corners to map pos
	local l_map_pos = v_div(l_corner,8)
	local r_map_pos = v_div(r_corner,8)

	-- check in the map if the value in both map pos are solid
	if(phy_is_solid(l_map_pos.x, l_map_pos.y) or
	   phy_is_solid(r_map_pos.x, r_map_pos.y)) then
		   obj.vel.y = 0
		   obj.pos.y = (flr(l_map_pos.y) - side) * 8
	end
	if(phy_is_building(l_map_pos.x, l_map_pos.y) or
	   phy_is_building(r_map_pos.x, r_map_pos.y)) then
			 obj.is_playing = false
			 obj.hit_building = true
	end
	if(phy_is_fire(l_map_pos.x, l_map_pos.y) or
	   phy_is_fire(r_map_pos.x, r_map_pos.y)) then
			 obj.is_playing = false
			 obj.hit_fire = true
	end
	if(phy_finish_level(l_map_pos.x, l_map_pos.y) or
	   phy_finish_level(r_map_pos.x, r_map_pos.y)) then
			 obj.is_playing = false
			 obj.finish = true
			 data_just_finished=true
			--  level_on += 1
	end
	if(phy_is_co2(l_map_pos.x, l_map_pos.y) or
	   phy_is_co2(r_map_pos.x, r_map_pos.y)) then
			 obj.is_playing = false
			 obj.hit_co2 = true
	end
	if(phy_is_macaw(l_map_pos.x, l_map_pos.y) or
	   phy_is_macaw(r_map_pos.x, r_map_pos.y)) then
			 obj.is_playing = false
			 obj.hit_macaw = true
	end
end

-- particle system
function ps_part_init(sprite, mass, life_time, system)
	local part    = obj_init(system.emitter, sprite, mass)
	part.grav     = false
	part.life_t   = life_time
	part.system   = system
	part.timer    = 0
	part.is_alive = false
	return part
end

function ps_part_update(part)
	if(part.is_alive) then
		update_physics(part)

		part.timer += 1
		if(part.timer > part.life_t) then
			part.is_alive = false
			part.timer = 0
			part.pos = part.system.emitter
			part.vel = v_init(0,0)
			part.acc = v_init(0,0)
		end
	end
end

function ps_part_draw(part)
	if(part.is_alive) then
		spr(part.s, part.pos.x, part.pos.y)
	end
end

function ps_part_shoot(part, f)
	part.is_alive = true
	apply_force(part, f)
end

function ps_system_init(sprite, pos, part_amount, shoot_freq)
	local system = {}
	system.emitter = pos
	system.particles = {}
	system.timer = 0
	system.shoot_freq = shoot_freq
	system.sprite = sprite

	for i=1,part_amount do
		local part = ps_part_init(sprite, 1, 30, system)
		add(system.particles, part)
	end

	return system
end

function ps_system_update(system, x_value, y_value, gravity)

	system.timer += 1
	if(system.timer > system.shoot_freq) then

		local part = nil

		-- let's pick the first particle alive in the system
		for i=1,#system.particles do
			if(not system.particles[i].is_alive) then
				part = system.particles[i]
				break
			end
		end

		if(part != nil) then
			local f = v_init(x_value, y_value)
			ps_part_shoot(part, f)
		end

		system.timer = 0
	end

	for i=1,#system.particles do
		ps_part_update(system.particles[i])
	end
end

function ps_system_draw(system)
	for i=1,#system.particles do
		ps_part_draw(system.particles[i])
	end
end

-- main game
function obj_move(obj,t)
  obj.anim_timer += 1
  if (obj.anim_timer == t) then
    obj.s = obj.anim_move[obj.anim_idx]
    obj.anim_idx += 1
    if (obj.anim_idx > #obj.anim_move) then
      obj.anim_idx = 1
    end
    obj.anim_timer = 0
  end
end

function obj_init(pos,s,mass, anim_move, col)
  local game_obj = {}
  game_obj.pos = pos
  game_obj.s = s
	game_obj.col = col
	game_obj.grav = true
  game_obj.vel  = v_init(0,0)
	game_obj.acc  = v_init(0,0)
	game_obj.mass = mass
  game_obj.anim_move = anim_move
  game_obj.anim_timer = 0
  game_obj.anim_idx = 1
	game_obj.is_flipped = false
	game_obj.is_on_ground = false
	game_obj.is_playing = true
	game_obj.hit_building = false
	game_obj.hit_fire = false
	game_obj.finish = false
	game_obj.off_screen = false
	game_obj.hit_co2 = false
	game_obj.hit_macaw = false
  return game_obj
end

function distance(x1,y1,x2,y2)
	local x_comp = x2-x1
	local y_comp = y2-y1
	local sqr = x_comp^2 + y_comp^2
 return sqrt(sqr)
end

function load_level_1()

	--delete if doesn't work
	if ogpos1 then
		player.pos = ogpos1
	end

	for x=0,map_width do
		for y=0,15 do
			local m_tile = mget(x,y)
			local pos = phy_map_to_screen(v_init(x,y))
			if(m_tile == 1) then
				local col = phy_aabb_init(0,0,8,8)
				if (change_sprite == "quetzal") then
					player = obj_init(pos,1,1,{1,2}, col)
				elseif (change_sprite == "toucan") then
					player = obj_init(pos,56,1,{56,57}, col)
				end
				ogpos1 = pos
				update_physics(player)
				mset(x,y,16)
		  elseif(m_tile == 3 or m_tile==32) then
				local col = phy_aabb_init(1,0,6,8)
			  avocado = obj_init(pos,3,1,{3}, col)
				add(avocados, avocado)
				mset(x,y,32)
			end
		end
	end
end

function load_level_2()

	--delete if doesn't work
	if ogpos2 then
		player.pos = ogpos2
	end

	for x=0,map_width do
		for y=16,31 do
			local m_tile = mget(x,y)
			local map = v_init(x,y)
			local pos = phy_map_to_screen(map)
			if(m_tile == 1) then
				local col = phy_aabb_init(0,0,8,8)
				player.pos=pos
				ogpos2 = pos
				update_physics(player)
				mset(x,y,16)
		  elseif(m_tile == 3 or m_tile==32) then
				local col = phy_aabb_init(1,0,6,8)
			  avocado = obj_init(pos,3,1,{3}, col)
				add(avocados, avocado)
				mset(x,y,32)
			end
		end
	end
end

level_on=1
avocado_count = 0
last_level = 2

change_sprite = "quetzal"
buy_toucan = false
buy_quetzal = false
own_toucan = false


function _init()
	map_width = 127
	map_height = 63
	s_pressed = false
	x_pressed = false
	x_just_pressed = false
	in_store = false
	made_sound1 = false
	made_sound2 = false
	if game_is_over == true then
		game_is_over = false
		s_pressed = true
	end
	game_is_over = false
  timer = 0
	macaw_timer = 0
	if level_on == 1 then
		fire_location = 484
	end
	if level_on == 2 then
		fire_location = 492
	end

	color1 = 4
	color2 = 4

	avocados = {}
	delete_avocados = {}

  y_adjust = (level_on - 1)*128

	macaw_pos = v_init(185,30 + 128)

	macaw = obj_init(macaw_pos,26,1, {26,28}, phy_aabb_init(1,0,13,16))

	menuitem(1,"debug mode",deb_start)

	if level_on==1 then
		load_level_1()
	end
	if level_on == 2 then
		load_level_2()
	end

	ps_fire_yellow = ps_system_init(4, v_init(fire_location,112 + y_adjust), 10, 1)
  ps_fire_orange = ps_system_init(5, v_init(fire_location,112 + y_adjust), 5, 1)
	ps_fire_red = ps_system_init(6, v_init(fire_location,112 + y_adjust), 5, 1)
	ps_fire_gray = ps_system_init(7, v_init(fire_location,112 + y_adjust), 15, 1)

	player.off_screen=false
	player.finish=false
	player.hit_building=false
	player.hit_fire=false
	player.hit_co2=false
	player.hit_macaw = false
	player.is_playing = true
end

function _update()
	if change_sprite == "toucan" then
		player.anim_move = {56,57}
	end
	if change_sprite == "quetzal" then
		player.anim_move = {1,2}
	end

	if game_is_over == false then
		if btn(0,1) then
			s_pressed = true
		end

		if s_pressed == true and not game_is_over then
			_update2()
		end
	end
	if game_is_over == true then
		if btn(4) then
			_init()
			camera(0, 0)
			cls()
		end
	end
	if btnp(5) then
		x_pressed = not x_pressed
		if x_pressed == false then
			in_store = false
		end
	end

end

function _update2()
	if player.is_playing then
		if in_store == false then
  	timer += 1
  end
		macaw_timer += 1
	end

	if timer > 672 and level_on == 1 then
		timer = 672
	end

	if timer > 895 and level_on == 2 then
		timer = 895
	end

--player movement

	if(btn(0)) then
		apply_force(player, v_init(-0.5, 0.0))
		player.is_flipped = true
	end

	if(btn(1)) then
		apply_force(player, v_init(0.5, 0.0))
		player.is_flipped = false
	end

	if(btn(2)) apply_force(player, v_init( 0.0,-0.5))

	if(btn(3)) apply_force(player, v_init(0,0.15))

  ps_system_update(ps_fire_yellow,-0.5 + rnd(1), -12 + rnd(12), false)
  ps_system_update(ps_fire_orange, -0.5 + rnd(1), -5 + rnd(5), false)
  ps_system_update(ps_fire_red, -0.25 + rnd(0.5), -1 + rnd(1), false)
  ps_system_update(ps_fire_gray, -1.5 + rnd(3), -20 + rnd(20), false)

	obj_move(player,10)
	update_physics(player)

	obj_move(macaw,10)
	update_physics(macaw)



	if macaw_timer <= 45 then
		apply_force(macaw, v_init(0,-.5))
	elseif macaw_timer > 45 and macaw_timer < 120 then
		return
	elseif macaw_timer == 120 then
		macaw_timer = 0
	end

	if player.pos.x < timer and not game_is_over then
		game_over(player)
		player.off_screen = true
	end

end

data_just_finished=false
function game_over(obj)
	cls()
	for n=0, 16 do
		for i=0, 16 do
			spr(12,timer + i*8 +1,n*8+ y_adjust)
		end
	end
	if obj.off_screen == true then
		print("you fell off the screen!", timer + 20,60 + y_adjust,7)
		print("press z to restart",timer + 30, 120 + y_adjust,6)
	end
	if obj.finish == true then
		if data_just_finished then
			level_on+=1
			data_just_finished=false
		end
		if level_on == last_level+1 then
			print("you won!", timer + 50,52 + y_adjust,7)
			print("thanks for saving the species",timer+2,60 + y_adjust)
			print("in the rainforest!",timer + 30, 68 + y_adjust)
		else
			print("you completed the level!",timer + 20,58 + y_adjust,7)
			print("press z to continue",timer + 30, 120 + y_adjust,6)
		end
	end
	if obj.hit_building == true then
		print("game over",timer + 40,44 + y_adjust,7)
		print("you hit a building!", timer + 30, 52 + y_adjust)
		print("when humans build in rainforest", timer + 3, 60 + y_adjust)
		print("habitats, habitat fragmentation" , timer + 3, 68 + y_adjust)
		print("threatens endangered species", timer + 10, 76 + y_adjust)
		print("press z to restart",timer + 30, 120 + y_adjust,6)
	end
	if obj.hit_fire == true then
		print("game over", timer + 40,44 + y_adjust,7)
		print("you touched the fire!", timer + 25,52 + y_adjust)
		print("forest fires can destroy entire", timer + 3, 60 + y_adjust)
		print("habitats, which puts all of the", timer + 3, 68 + y_adjust)
		print("endangered species at risk.", timer + 10, 76 + y_adjust)
		print("press z to restart",timer + 30, 120 + y_adjust,6)
	end
	if obj.hit_co2 == true then
		print("game over", timer + 40,44 + y_adjust,7)
		print("you touched the carbon dioxide!", timer + 4, 50 + y_adjust)
	  print("co2 is one of the greenhouse ",timer + 5,56 + y_adjust)
		print("gases that causes heat to be ",timer + 5,62 + y_adjust)
		print("trapped, which increases the ",timer + 5,68 + y_adjust)
		print("surface temperature through", timer + 7, 74 + y_adjust)
		print("global warming", timer + 30,80 + y_adjust)
		print("press z to restart",timer + 30, 120 + y_adjust,6)
	end
	if obj.hit_macaw == true then
		print("game over", timer + 40,44 + y_adjust,7)
		print("you hit the macaw!", timer + 30, 52 + y_adjust)
		print("other birds in the rainforest ",timer + 4,60 + y_adjust)
		print("serve as competition towards ", timer + 5,68 + y_adjust)
		print("endangered species that search", timer +4, 76 + y_adjust)
		print("for similar resources",timer + 25, y_adjust + 84)
		print("press z to restart",timer + 30, 120 + y_adjust,6)
	end
end

function store()
	cls()
	in_store = true
	camera(800,0)
	map(0,0,0,0,128,64)
	timer = 0
	rect(831,8,864,32,color1)
	rect(831,56,864,80,color2)
	print("select",837,19,7)
	print("select",837,67,7)
	print("the quetzal is",868,14,7)
	print("an endangered ",868,20)
	print("bird from cloud ", 868,26)
	print("forests that ",868,32)
	print("eats avocados",868,38)

	print("toucans are a",868,52,7)
	print("threatened bird",868,58)
	print("species in the", 868,64)
	print("rainforest that",868,70)
	print("eat various",868,76)
	print("fruits",868,82)

	if btn(3) then
		color1 = 4
		color2 = 7
	end
	if btn(2) then
		color1 = 7
		color2 = 4
	end
	if color1 == 7 and btn(1,1) then
		cant_buy = false
		player.s = 1
		change_sprite = "quetzal"
		buy_quetzal = true
		buy_toucan = false
		sfx(1)
	end
	if color2 == 7 and btn(1,1) then
		if buy_toucan == false then
			if own_toucan == false then
				if avocado_count >= 30 then
					cant_buy = false
					buy_toucan = true
					buy_quetzal = false
					change_sprite = "toucan"
					avocado_count -= 30
					sfx(2)
					own_toucan = true
				else
					cant_buy = true
					buy_toucan = false
					buy_quetzal = false
			  end
			else
				cant_buy = false
				buy_toucan = true
				buy_quetzal = false
				change_sprite = "toucan"
				sfx(1)
				own_toucan = true
			end
		end
	end
	if buy_toucan == true then
		print("you selected the toucan!", 820,96,7)
	end
	if buy_quetzal == true then
		print("you selected the quetzal!", 820,96,7)
	end

	if cant_buy == true then
		print("you don't have enough tokens", 815,96,7)
	end

	print(" = 30", 816,82,7)
	print(" = 0", 816,34,7)
	print("press x to exit",838,104,6)
	print("use arrows to select",830,112,6)
	print("press f to buy",840,120,6)
	spr(3,896,2)
	print(" = "..avocado_count, 904,4)
	rect(895,0,926,11,1)
	player.pos = v_init(30,60 + y_adjust)

end

function collect_avocados()
	for i=1,#avocados do

		if(phy_aabb_collide(player.pos, player.col, avocados[i].pos, avocados[i].col) == true) then
			--mset(avocados[i].pos.x,avocados[i].pos.y,12)
			add(delete_avocados, avocados[i])
			sfx(0)
			avocado_count += 1
		end
	end

	for i=1, #delete_avocados do
		del(avocados,delete_avocados[i])
	end
end

function obj_draw(obj)
	spr(obj.s, obj.pos.x, obj.pos.y, 1, 1, obj.is_flipped, false)
end

function obj_draw2(obj)
	spr(obj.s, obj.pos.x, obj.pos.y, 2, 2, obj.is_flipped, false)
end

function _draw()
	cls()
		if s_pressed == false then

			if player.off_screen==true then
				player.off_screen=false
			end
			if player.is_playing==false then
				player.is_playing=true
			end
			if game_is_over==true then
				game_is_over=false
			end
				for n=0, 15 do
					for i=0, 15 do
						spr(12,timer + i*8,n*8)
					end
				end
			  print("press s to start",30,5,7)
				print("rainforest rescue!",29.75,60,3)
				print("rainforest rescue!",30,60,11)
		else
			_draw2()
		end
end

function draw_level_1()
	for n=0, 14 do
		for i=0, 15 do
			spr(12,timer + i*8-1,n*8 + y_adjust)
		end
	end
	map(0,0,0,0,128,64)
	--fire particle system
	spr(8,fire_location,112 + y_adjust)
	spr(9,fire_location,104 + y_adjust)

	ps_system_draw(ps_fire_gray)
	ps_system_draw(ps_fire_yellow)
	ps_system_draw(ps_fire_orange)
	ps_system_draw(ps_fire_red)

	foreach(avocados, obj_draw)
	obj_draw(player)
end

function draw_level_2()
	for n=0, 14 do
		for i=0, 15 do
			spr(12,timer + i*8,n*8 + y_adjust)
		end
	end
	map(0,0,0,0,128,64)

	--fire particle system
	spr(8,fire_location,112 + y_adjust)
	spr(9,fire_location,104 + y_adjust)

	ps_system_draw(ps_fire_gray)
	ps_system_draw(ps_fire_yellow)
	ps_system_draw(ps_fire_orange)
	ps_system_draw(ps_fire_red)

	obj_draw(player)

	foreach(avocados, obj_draw)
	obj_draw2(macaw)
end

function _draw2()
  cls()
	if x_pressed == true then
		store()
	else
		collect_avocados()
		camera(timer, y_adjust)
		if level_on == 1 then
			draw_level_1()
		end
		if level_on == 2 then
			draw_level_2()
		end

		print("press x for store",timer + 30, y_adjust + 121,6)
		rectfill(90 + timer,y_adjust,127 + timer ,10+ y_adjust,2)
		rect(90 + timer,y_adjust,127 + timer ,10+ y_adjust ,6)
		spr(3,95 + timer, 1+ y_adjust)
		print(" = "..avocado_count,105 + timer,3+ y_adjust,7)
		print("level "..level_on,timer + 1,y_adjust+1)
		print(timer,timer+1,10 + y_adjust)

		if player.pos.x < timer then
			player.is_playing = false
			game_is_over = true
			player.off_screen = true
		end

		if player.is_playing == false then
			game_over(player)
			game_is_over = true

			if made_sound1 == false and made_sound2 == false then
				if player.finish == false then
					sfx(3)
					made_sound1 = true
				else
					sfx(4)
					made_sound2 = true
				end
			end
		end

		for i=1, #deb_log_buffer do
			print(deb_log_buffer[i], timer + 5, y_adjust + (i*5), 2)
	 	end
	end
end
__gfx__
000000000000bbb0b000bbb000033000000000000000000000000000000000009999999900000000000a000033333333cccccccccccccccccccccccc77557755
000000000000b5babb00b5ba00333300000000000000000000000000000000009989999900000000000a000033333333cccccccccccccccccccccccc77557755
007007000000bbb0bbb0bbb0033bb33000000000009999000088880000066600898989989000900900aaa00033333333cccccccccccccccccccccccc55775577
00077000000bb8800bbbb88003bbbb30000aa00000999900008888000006660089898898990090090aaaaa0033333333cccccccccccccccccccccccc55775577
00077000000bb88000bbb88003b44b30000aa000009999000088880000066600888988889990990900aaa00044444444cccccccccccccccccccccccc77557755
0070070000bbb880000bb88003b44b30000000000099990000888800000000008888888899999999000a000044444444cccccccccccccccccccccccc77557755
000000000bbb88000bbb8800033bb330000000000000000000000000000000000888888099999999000a000044444444cccccccccccccccccccccccc55775577
00000000bb000000bb000000003333000000000000000000000000000000000000888800999999990000000044444444cccccccccccccccccccccccc55775577
000000000000bbb00000bbb000000000555555555555555500000000600600600000006000000000008880000abd000000888000000000004444444400000000
000000000000b5ba0000b5ba00000000555555555555555500000000000000066660000000000000078780008abd000007878000000000004444444400000000
000000000000bbb00000bbb000000055555555555555555550000000606600666666606000000000008880088abd000000888000000000004444444400000000
00000000000bb880000bb8800000005555555555555555555000000006666666666660000000000000088088aabd000000088800000000004444444400000000
00000000000bb880000bb880000000555555555555555555500000006665556555660006000000000008888aabdd000000088a80000000004444444400000000
0000000000bbb88000bbb8800000005555aa555aa555aa5550000000666566656566066000000000000888aabdd000000008baa8000000004444444400000000
000000000bbb88000bbb88000000005555aa555aa555aa5550000000066566656566666000000000000888abdd0000000008dba8000000004444444400000000
00000000bb0a0000bb000a000000005555aa555aa555aa5550000000066566656566666000000000000888bdd00000000008ddba000000004444444400000000
00000000444444444bbbb4440000005555aa555aa555aa5550000000666566656565566000000000000088dd0000000000008ddba00000004444444400000000
0000000044444444bbbbbb44000000555555555555555555500000006665666565666566000000000000888800000000000088ddba0000005555555500000000
0000000044444444bb00bba40000005555555555555555555000000066655565556656660000000000060888800000000006088dda0000005555555500000000
0000000044444444bb00bba400000055555555555555555550000000066666666665666000000000000060000800000000006008880000005555555500000000
0000000044444444bbbbbb440000005555aa555aa555aa5550000000066666666665556000000000000000000080000000000000080000005555555500000000
000000004444444bbbbbb4440000005555aa555aa555aa5550000000006666600666666000000000000000000008000000000000008000005555555500000000
000000004444444bbb8884440000005555aa555aa555aa5550000000600006000606660000000000000000000000800000000000000800005555555500000000
00000000444444bbb88888440000005555aa555aa555aa5550000000000600006000000600000000000000000000080000000000000080004444444400000000
0000000044444bbbb888884400000055555555555555555550000000444334440005599950055999000559990000000000000000000000000000000000000000
000000004444bbbbb888884400000055555555555555555550000000443333440005199955051999000509990000000000000000000000000000000000000000
000000004444bbbbb888884400000055555555555555555550000000433bb3340005550955555509000555090000000000000000000000000000000000000000
00000000444bbbbbb88884440000005555aa555aa555aa555000000043bbbb340055500005555000005550000000000000000000000000000000000000000000
0000000044bbbbbbb88444440000005555aa555aa555aa555000000043b44b340055500000557000005570000000000000000000000000000000000000000000
000000004bbb4bbbb84444440000005555aa555aa555aa555000000043b44b340055550000557000005570000000000000000000000000000000000000000000
00000000bbb444a44aa444440000005555aa555aa555aa5550000000433bb33405555550055500000555a0000000000000000000000000000000000000000000
00000000bb4444aa4444444400000055555555555555555550000000443333445550050055500000050000000000000000000000000000000000000000000000
00000000000000000000000000000055555555555555555550000000444444444444444400000000000000000000000000000000000000000000000000000000
00000000000000000000000000000055555555555555555550000000445555544444444455000000000000000000000000000000000000000000000000000000
0000000000000000000000000000005555aa555aa555aa55500000004555aaabbbbbbb4455500000555500000000000055550000000000000000000000000000
0000000000000000000000000000005555aa555aa555aa55500000004555a5a999999984555500055aabbb88000000055aabbb88000000000000000000000000
0000000000000000000000000000005555aa555aa555aa55500000004555aaabbbbbb98466555005a5a9999800000005a5a99998000000000000000000000000
0000000000000000000000000000005555aa555aa555aa55500000004555aaa44444488406655505aaaa000800000005aaaa0008000000000000000000000000
000000000000000000000000000000555555555555555555500000004455aaa44444448400665555aaa0000000000055aaa00000000000000000000000000000
000000000000000000000000000000555555555555555555500000004455aaa44444444400065555555000000000055555500000000000000000000000000000
0000000000000000000000004444444444444444444444444444444445555aa54444444400066555550000000000555555000000000000000000000000000000
00000000000000000000000044444444444444444444444444444444455555555444444400005555500000000000556555500000000000000000000000000000
00000000000000000000000044444444444444444444444444444444455555555444444400005555500000000000555655550000000000000000000000000000
00000000000000000000000044444444444444444444444444444444445555555444444400055555000000000005555665555000000000000000000000000000
00000000000000000000000044444555555555555555555555544444445555554444444455555000000000005555500006555000000000000000000000000000
000000000000000000000000444555dddddddddddddddddddd555444444f55f44444444405550000000000000555000000655000000000000000000000000000
0000000000000000000000004455dddddddddddddddddddddddd554444ff44ff4444444400550000000000000055000000055000000000000000000000000000
000000000000000000000000455dddddddddddddddddddddddddd554444444444444444400050000000000000005000000005000000000000000000000000000
000000000000000000000000455ddddddddddddddddddddddddddd54000000000000000000000000000000000000000000000000000000000000000000000000
00000000000000000000000055dddddddddddddddddddddddddddd55000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000005dddddddddddddddddddddddddddddd5000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000005dddddddddddddddddddddddddddddd5000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000005dddddddddddddddddddddddddddddd5000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000005dddddddddddddddddddddddddddddd5000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000005dddddddddddddddddddddddddddddd5000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000005dddddddddddddddddddddddddddddd5000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000005dddddddddddddddddddddddddddddd5000000000000000000000000000000000000000000000000000000000000000000000000
00000000000000000000000055dddddddddddddddddddddddddddd55000000000000000000000000000000000000000000000000000000000000000000000000
000000000000000000000000455ddddddddddddddddddddddddddd54000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000004455ddddddddddddddddddddddddd554000000000000000000000000000000000000000000000000000000000000000000000000
000000000000000000000000444555dddddddddddddddddddddd5544000000000000000000000000000000000000000000000000000000000000000000000000
000000000000000000000000444455dddddddddddddddddddd555444000000000000000000000000000000000000000000000000000000000000000000000000
00000000000000000000000044444555555555555555555555544444000000000000000000000000000000000000000000000000000000000000000000000000
00000000000000000000000044444444444444444444444444444444000000000000000000000000000000000000000000000000000000000000000000000000
d0d0d0d0d0d0d0d0d0d0d0d044444444444444444444444444444444d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0
d0d0d0d0d0d0d0d0d0d0d0d044444444444444444444444444444444d0d0d0d0d0d0d0f001000000000000000000000000000000000000000000000000000000
91919191919191910101010144444444444444444444444444444444010101010101019191919191f1f100000000000000000000000000000000000000000000
0000000000000000000000004444444444444444444444444444444400000000000000f001000000000000000000000000000000000000000000000000000000
919191919191919191919191444444444444444444444444444444440101010101010101919191f1f1f1f1919191000091009100000091919100000000000000
0000000000000000000000004444444444444444444444444444444400000000000091f001000000000000000000000000000000000000000000000000000000
919191919191919191919191444444444444444444444444444444449191919191910101919191f1f1f1f1919191919191919100009100919100000000000000
0000000071810000000000004444444444444444444444444444444400000000000091f001000000000000000000000000000000000000000000000000000000
91919191919191919191919191919130919191309191309191919191919191919191919191919191000000000000000091919100000000919100000000000000
0000000072820000000000718100000001010101010000000000000000000000000091f001000000000000000000000000000000000000000000000000000000
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
0000000000000000910000728200000001010101000000000000000000000000009191f001000000000000000000000000000000000000000000000000000000
91919191919191919191919191919191919191919191919130919191919191919191919191919191000000000071810000000000000000000000000000000000
3000000000910001910000000000010000000000000000000000000000000000009191f001000000000000000000000000000000000000000000000000000000
91919191919191919191919191919191919191919191919191919191919191919191913141516191000000000072820000000071810000009100000000000000
0000000000000030000000000000300000314151610000000000000000000000919191f001000000000000000000000000000000000000000000000000000000
919191919110919191919191919191919191919191919191919130919191919191919132425262910001000000000000000000728200000091910000e0e00000
0000000000000000000000000000000000324252620000000000000000000000910091f001000000000000000000000000000000000000000000000000000000
919191919191919191919191919191919191919191919191919191919191919191919133435363910001010000000000000000000000000091910000e0e00000
0000000000000091910000000000000000334353630000000000000000000000000000f001000000000000000000000000000000000000000000000000000000
919191919191919191919191919191919191919191919191919191913091913091919134445464910101010000003000003000000000000091910000e0e00000
0000000000007181910000000000718100344454640000000000000000000000000000f001000000000000000000000000000000000000000000000000000000
919191919191919191919191919191919191919191919191919191919191919191919133435363910101000000010100000000000000000000000000e0e00000
0000000000007282000000000000728200334353630000000000000000000000000000f001000000000000000000000000000000000000000000000000000000
919191919191919191919191919191919191919191919191919191919191919191919134445464910101003000009100009100003000003000718100e0e00000
0000003000009100009130000000000000344454640000000000000000000000000000f001000000000000000000000000000000000000000000000000000000
919191919191919191919191919191919191919191919191919191919191919191919133435363910101000000000071810000000000000000728201e0e00000
0000000000009100000000000000000000334353630100000000000000000000000000f001000000000000000000000000000000000000000000000000000000
919191919191919191919191919191919191919191919191919191919191919191919134445464910100000000000072829191919191919101010101e0e00000
0000000000000000000000000000000000344454640101010000000000000000000000f001000000000000000000000000000000000000000000000000000000
b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0
b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0f001010100000000000000000000000000000000000000000000000101
d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0
d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0f0
91910101010101010101010101010101010101010101919191919191919191919191919101019191000000000000000000000000000000000000000000000000
010101010100000000010101010100000000000000000000010101000000000000000001010101010101000000000000000000000000000000000001010101f0
9191919191919191919191919191919191e1d1919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000101010101010000000000000000000000000000010001010101010101010101000100000000000000000000000000000000000000000000000000f0
9191919191919191919191919191919191e1e1919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191109191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191919191919191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919191910101019191919191919191919191919191000000000000000000000000000000000000000000000000
000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000f0
91919191919191919191919191919191919191919101010101019191919101010101010101010101000000000000000101010100000000000000000001010000
000000000000000000000000000000000000000000000000000000000101010101000000000000000000000001000000000000000000000000000000000000f0
b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0
b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0f0
__label__
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddd000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd00000000
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddfdddddddddddddfdddddfdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddd000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd00000000000000000000000000000000
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddd000000000000000000000000000000000000000000000000000000000000000000000000000000000000
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd00000000000000000000000000000000000000000000
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd

__gff__
0000000000000000000000010001040800000000020200101000202020000000000000000202001010002020202000000000000002020000000000000000000000000000020200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
__map__
0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0f1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e000000000000000000000000
191919191919191910101010101010101010101010101010101010101010101010101019191919191f1f0000000000000000000000000000000000000000000000000000000000000000000000000000000000000010101010000000000000000000000f1e21221e535455561e1e1e1e1e1e1e1e000000000000000000000000
191919191919191919191919191910101010101019191910101010101010101010101010191917181f1f1f19191900001900190000001919190000000000000000000000000000000000000000000000101010000010101010100000000000000000190f1e31321e636465661e1e1e1e1e1e1e1e000000000000000000000000
19191919191919191919191919191919191919191919191919191919191919191919101019192728101f1f19191919191919190000190019190000000000000000000000171800000000000000000000101010101000001010100000000000000000190f1e1e1e1e737475761e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919191919191919190319191903191903191919191919191919191919191919101010100000000000001919190000000019190000000000000000000000272800000000001718000000101010101000000000000000000000000000190f1e371e1e838485861e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919101019101919191919191919191919191919191919191919191919191919191900000000000000000000000000000000000000000000000000000000000000001900002728000000101010100000000000000000000000000019190f2e2e2e2e838485861e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919101019101919191919191919191919190319191919191919191919191919191900000000001718000000000000000000000000000000000003000000001900101900000000001000000000000000000000000000000000000019190f1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919101019101919191919191919191919191919191919191919191919131415161900000000002728000000001718000000190000000000000000000000000000030000000000000300001314151600000000000000000000001919190f1e47481e535455561e1e1e1e1e1e1e1e000000000000000000000000
1919191919011919191010101919191919191919191919191919031919191919191919232425261900100000000000000000002728000000191900000e0e000000000000000000000000000000000000002324252600000000000000000000001900190f1e57581e636465661e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919191019101919191919191919191919191919191919191919191919333435361900101000000000000000000000000000191900000e0e000000000000000000191900000000000000003334353600000000000000000000000000000f1e1e1e1e737475761e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919191919191919191919191919191919191919191903191903191919434445461910101000000003000003000000000000191900000e0e000000000000000017181900000000001718004344454600000000000000000000000000000f1e371e1e838485861e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919191919191919191919191919191919191919191919191919191919333435361910100000001010000000000000000000000000000e0e000000000000000027280000000000002728003334353600000000000000000000000000000f2e2e2e2e2e1e1e1e1e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919191919191919191919191919191919191919191919191919191919434445461910100003000019000019000003000003001718000e0e000000000003000019000019030000000000004344454600000000000000000000000000000f1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919191919191919191919191919191919191919191919191919191919333435361910100000000000171800000000000000002728100e0e000000000000000019000000000000000000003334353610000000000000000000000000000f1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e000000000000000000000000
1919191919191919191919191919191919191919191919191919191919191919191919434445461910000000000000272819191919191919101010100e0e000000000000000000000000000000000000004344454610101000000000000000000000000f1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e000000000000000000000000
0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0f1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e000000000000000000001010
0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0d0f
191910101010101010101010101010101010101010101919191919191919191919191919101019190000000000171800000000000000000000000000000000001010101010000000001010101010000000000000000000001010100000000000000000101010101010100000000000000000000000000000000000101010100f
1919191919191919191919191919191919101d1919171819191718191919191919191919191919190000000010272800000000000000171800000000001010000000000010101017181000000000000000000000000000001000101010101010101010100010000000000000000000000000000000000000000000000000000f
191919191919191919190319191903191910101919272819192728191919171819191919191919190000001718101010000000000000272800000017181010100000000300000027280000000000000000001718000000000000030000000000000000000000000000000000000000000000000000000000000000000000000f
191919191919191919191919191919191919191919191919191919191919272819191919190319190000002728000000000003000000000000001027280000001000000000000000000000000000000000002728000000000000000000000000000000000000000003000000000000000017181000000000000000000000000f
191919191919191910101919191919191919191919191919191919191919191919191919191919190000001010000000000000000000000000101010100000101010000000000000000000000000000000000000000000000000000000000000131415160000000000000000000000001027281000000000000010000000000f
191919191919191910101919191919191919191919191019191919190319191919191919191919190000000000000000000000000000101010101010000000001010171800000000000000000017180000000000000000000000000000000000232425260000000000000000000000001010101000000003000000000000000f
191919190119191910101919191718191919191919191919191919191919191919191919191917180000000000000000000000000003000000000000000000000000272800000003000000000027280000000000030000001718000000030000333435360000000000000000000000001010101010001000000000000000000f
191919191919191910191919192728191903191919191919191919191919191919190319191927280000000013141516000000000000000000000000000e0e000000000000000000000000000000000000000000000000002728000000000000434445460000000000171800000000000000100000001000000000030000000f
191919191919191919191919191919191919191919191919191010191919191919191919191919190000000023242526001718000000000000000000000e0e000000000000000000000000000000000000000000000000000000000000000000333435360000000000272800000003000000100017180000000000000010000f
191919191919191919191919191919191919191919191919191919191919191919191919191919190000000033343536002728000000000000000000000e0e000000000000000000000000000000000000000000000000000000000000000000434445460000000000000000000000000000100027280000000000000000000f
191919191919191917181919191919191919191919191919191919191718191919191919191919190000000043444546000000000000000003000000000e0e000000001718000000000000030000000000000300000000000000000000000000434445460000000000000000000000000000101000000000000000000000000f
191919191919191927281919191919191919191903191919191919192728191919191019191910100000000043444546000000001718000000000000000e0e000000002728000000000000000000000000000000000000000000000000000000333435360010000000000000000000001718101000000000000000000000000f
191919191919191919191919191919191919191919191910101019191919191919191919191919190000000033343536000000002728000000000000000e0e100000000000000000000000000000000000000000000000000000000000000000434445460000000000000000000000002728101000000000000000000000000f
191919191919191919191919191919191919191919101010101019191919101010101010101010100000000043444546101010000000000000000000100e0e000000000000000000000000000000000000000000000000000000000010101010434445460000000000000000100000000000000000000000000000000000000f
0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0f
__sfx__
0003000020750237502575026750287502a7502c7502c75021700257002a7002c700327002a0002f0001a00016000110000e0000c000000000000000000000000000000000000000000000000000000000000000
000300000e000000000f000133001330012000163001f0202102025020280202a0202d02030020350203802013000130000000013000170001300013000130001140000000000000000000000000000000000000
000400000000000000000000000000000000000000000000202002320024330283302f33027330183301a330193301d330253302b33000000000001720017200192001b200000001d2001e2001e2000000000000
00070000000002d3402834024340203401c3401a34017340143401534014340133401334012340053000530005300043000430004300043001e3002310024100183001530026100113000d3000c3000000000000
00060000170301e0302303024030390001803020030270302c0302e030310001c03029030310303303033030240003105033050340503505036050360003700037000390003a0000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
__music__
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
00 41424344
