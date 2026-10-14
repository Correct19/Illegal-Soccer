local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local GuiService = game:GetService("GuiService")
local SoundService = game:GetService("SoundService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local NOME_BOLA = "Ball"
local NOME_JOGADOR = "gdheohkkqegw"
local JERSEY_MIN = 235
local FORCAR_GOL = nil

local GK_ZONA_DIAMETRO = 75
local CERTEZA_DIAMETRO = 18
local BOLA_DIAMETRO = 10
local MARCADOR_DIAMETRO = 2
local GK_ZONA_TRANSP = 0.88
local CERTEZA_TRANSP = 0.88
local CERTEZA_TRANSP_PRONTA = 0.7
local BOLA_TRANSP = 0.75
local ALCANCE_BOLA = (CERTEZA_DIAMETRO + BOLA_DIAMETRO) / 2

local MARGEM_TRAVE = 0.8
local CANTO_PADRAO = 9
local VEL_CHUTE_ESTIMADA = 90
local PREFERE_TRAVE_LONGE = 0.25
local GK_MAX_DIST_GOL = 45
local JANELA_MIRA = 0.7
local RECALCULAR_A_CADA = 0.1
local FORCA_MIRA = 0.8

local CURVA_ATIVA = true
local INVERTER_CURVA = false
local CURVA_DESVIO = 3
local SLIDE_ESCALA = 0.9
local SLIDE_PASSOS = 3
local SLIDE_DURACAO = 0.05
local CURVA_COOLDOWN = 0.6

local PREFIXO_SOM = "CharacterTackle"
local DISTANCIA_TACKLE = 10
local SO_COM_BOLA = true
local DISTANCIA_BOLA = 7
local IGNORAR_MEU_TIME = true
local ATRASO_CLIQUE = 0
local DRIBLE_COOLDOWN = 1

local FAKE_SEGURAR = 0.2
local ANTECEDENCIA = 0.001
local SOLTAR_CHUTE_DEPOIS = 0.05
local RAIO_PRESSAO = 14
local PRESSAO_MIN = 1.2
local CONFIRMAR = 2
local INTERVALO_ANALISE = 0.1
local BOLA_PERTO = 7
local FAKE_COOLDOWN = 2.5
local SO_NO_ATAQUE = true
local DIST_GOL_MAX = 110
local CONTAR_GOLEIRO = false
local CONTAR_SEM_CAMISA = false

local ID_BASE = 50000
local DURACAO_TOQUE = 0.05
local PAUSA_USUARIO = 1
local HOLD_REAL_MAX = 1.5

local MOSTRAR_CHAVES = true
local DEBUG = true

local function log(...)
	if DEBUG then
		print("[Atacante]", ...)
	end
end

local ultimosLogs = {}
local function logLento(chave, ...)
	local agora = os.clock()
	if agora - (ultimosLogs[chave] or 0) >= 0.5 then
		ultimosLogs[chave] = agora
		log(...)
	end
end

local okVIM, VirtualInputManager = pcall(function()
	return game:GetService("VirtualInputManager")
end)

local mobile = playerGui:WaitForChild("Mobile", 10)
local frame = mobile and mobile:WaitForChild("Frame", 10)
local botao0 = frame and frame:WaitForChild("Button0", 10)
local botao2 = frame and frame:WaitForChild("Button2", 10)

if not (botao0 and botao2) then
	warn("Button0 ou Button2 não encontrado")
	return
end

local caracteres = workspace:WaitForChild("Characters", 10)
local playersFolder = caracteres and caracteres:WaitForChild("Players", 10)
local npcsFolder = caracteres and caracteres:FindFirstChild("NPCs")

local ativo = { mira = true, drible = true, fake = true }

local gui = Instance.new("ScreenGui")
gui.Name = "SistemaAtacante"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.Parent = playerGui

local function criarChave(chave, texto, x)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(0, 62, 0, 26)
	b.Position = UDim2.new(0, x, 0, 8)
	b.BorderSizePixel = 0
	b.TextColor3 = Color3.new(1, 1, 1)
	b.TextSize = 12
	b.Font = Enum.Font.GothamBold
	b.Text = texto
	b.Visible = MOSTRAR_CHAVES
	b.Parent = gui

	local function atualizar()
		b.BackgroundColor3 = ativo[chave] and Color3.fromRGB(40, 150, 80) or Color3.fromRGB(170, 50, 50)
		b.BackgroundTransparency = 0.25
	end
	atualizar()

	b.Activated:Connect(function()
		ativo[chave] = not ativo[chave]
		atualizar()
		log(texto, ativo[chave] and "ligado" or "desligado")
	end)
end

criarChave("mira", "MIRA", 8)
criarChave("drible", "DRIBLE", 76)
criarChave("fake", "FAKE", 144)

local function pegaGol(nome)
	local m = workspace:FindFirstChild("Map")
	local d = m and m:FindFirstChild("Data")
	local t = d and d:FindFirstChild(nome)
	return t and t:FindFirstChild("Goal")
end

local function infoGol(g)
	if g:IsA("BasePart") then
		return g.CFrame, g.Size
	end
	local ok, cf, size = pcall(function()
		return g:GetBoundingBox()
	end)
	if ok then
		return cf, size
	end
	return CFrame.new(), Vector3.new(0, 0, 0)
end

local function achaHandle(modelo)
	local camisa = modelo and modelo:FindFirstChild("TeamJersey")
	if not camisa then
		return nil
	end
	if camisa:IsA("BasePart") then
		return camisa
	end
	return camisa:FindFirstChild("Handle", true)
end

local function ehBranca(c)
	return c.R * 255 >= JERSEY_MIN and c.G * 255 >= JERSEY_MIN and c.B * 255 >= JERSEY_MIN
end

local function corParecida(a, b)
	local dr, dg, db = a.R - b.R, a.G - b.G, a.B - b.B
	return dr * dr + dg * dg + db * db < 0.0225
end

local function meuModelo()
	return (playersFolder and playersFolder:FindFirstChild(NOME_JOGADOR)) or player.Character
end

local function minhaCor()
	local h = achaHandle(meuModelo())
	if h and h:IsA("BasePart") then
		return h.Color
	end
	return nil
end

local function souGoleiro()
	local c = minhaCor()
	return c ~= nil and ehBranca(c)
end

local modoGoleiro = false

task.spawn(function()
	while true do
		modoGoleiro = souGoleiro()
		task.wait(0.25)
	end
end)

local bolas = {}

local function getParte(obj)
	if obj:IsA("BasePart") then
		return obj
	end
	if obj:IsA("Model") then
		return obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart", true)
	end
	return nil
end

local function registrarBola(obj)
	if bolas[obj] then
		return
	end
	local p = getParte(obj)
	if not p then
		return
	end
	bolas[obj] = p
	obj.AncestryChanged:Connect(function()
		if not obj:IsDescendantOf(workspace) then
			bolas[obj] = nil
		end
	end)
end

local function bolaMaisProxima(pos)
	local melhor, menor
	for obj, p in pairs(bolas) do
		if p.Parent then
			local s = p.Size
			if math.max(s.X, s.Y, s.Z) >= 1.5 then
				local d = (p.Position - pos).Magnitude
				if not menor or d < menor then
					menor = d
					melhor = p
				end
			end
		else
			bolas[obj] = nil
		end
	end
	return melhor, menor
end

local idToque = ID_BASE
local virtualAte = {}
local gestoAtivo = false
local fakeAtivo = false

local function novoId()
	idToque += 1
	return idToque
end

local function marcarVirtual(botao, segundos)
	virtualAte[botao] = math.max(virtualAte[botao] or 0, os.clock() + segundos)
end

local function ehVirtual(botao)
	return os.clock() < (virtualAte[botao] or 0)
end

local function enviarToque(id, estado, botao, dx)
	if not okVIM then
		return false
	end
	local c = botao.AbsolutePosition + botao.AbsoluteSize / 2
	local inset = GuiService:GetGuiInset()
	return pcall(function()
		VirtualInputManager:SendTouchEvent(id, estado, c.X + (dx or 0), c.Y + inset.Y)
	end)
end

local function tocarBotao(botao)
	if not okVIM then
		return false
	end
	local id = novoId()
	marcarVirtual(botao, DURACAO_TOQUE + 0.25)
	local ok = enviarToque(id, 0, botao)
	task.wait(DURACAO_TOQUE)
	enviarToque(id, 2, botao)
	return ok
end

local function apertarDribbleMultiplo()
	if not okVIM then
		local disparou = false
		pcall(function()
			for _, c in ipairs(getconnections(botao0.Activated)) do
				c:Fire()
				disparou = true
			end
			for _, c in ipairs(getconnections(botao0.MouseButton1Click)) do
				c:Fire()
				disparou = true
			end
		end)
		return disparou
	end

	task.spawn(function()
		local id = novoId()
		marcarVirtual(botao0, DURACAO_TOQUE + 0.1)
		enviarToque(id, 0, botao0)
		task.wait(DURACAO_TOQUE)
		enviarToque(id, 2, botao0)
	end)
	return true
end

local function deslizar(lado)
	if not okVIM or gestoAtivo then
		return false
	end
	gestoAtivo = true
	local id = novoId()
	local dist = math.max(40, botao2.AbsoluteSize.X * SLIDE_ESCALA) * lado
	marcarVirtual(botao2, SLIDE_DURACAO + 0.35)

	task.spawn(function()
		pcall(function()
			enviarToque(id, 0, botao2, 0)
			for i = 1, SLIDE_PASSOS do
				task.wait(SLIDE_DURACAO / SLIDE_PASSOS)
				enviarToque(id, 1, botao2, dist * i / SLIDE_PASSOS)
			end
		end)
		enviarToque(id, 2, botao2, dist)
		task.delay(0.1, function()
			gestoAtivo = false
		end)
	end)
	return true
end

local ultimoToqueUsuario = 0
local inicioHoldReal2 = nil

local function usuarioOcupado()
	local agora = os.clock()
	if agora - ultimoToqueUsuario < PAUSA_USUARIO then
		return true
	end
	if inicioHoldReal2 and agora - inicioHoldReal2 < HOLD_REAL_MAX then
		return true
	end
	return false
end

local function pontuacaoOlhar(raiz, gol)
	local olhar = Vector3.new(raiz.CFrame.LookVector.X, 0, raiz.CFrame.LookVector.Z)
	local p = infoGol(gol).Position
	local dir = Vector3.new(p.X - raiz.Position.X, 0, p.Z - raiz.Position.Z)
	if olhar.Magnitude < 0.01 or dir.Magnitude < 0.01 then
		return 0
	end
	return olhar.Unit:Dot(dir.Unit)
end

local function escolheGol(raiz)
	if FORCAR_GOL then
		return pegaGol(FORCAR_GOL)
	end
	local g1, g2 = pegaGol("Team1"), pegaGol("Team2")
	if not (g1 and g2) then
		return g1 or g2
	end
	return pontuacaoOlhar(raiz, g1) >= pontuacaoOlhar(raiz, g2) and g1 or g2
end

local function achaGoleiro(golPos)
	local meu = meuModelo()
	local melhor, menor = nil, math.huge

	local function avaliar(modelo)
		if modelo == meu or modelo.Name == NOME_JOGADOR then
			return
		end
		local h = achaHandle(modelo)
		if h and h:IsA("BasePart") and ehBranca(h.Color) then
			local r = modelo:FindFirstChild("HumanoidRootPart") or modelo.PrimaryPart
			if r then
				local d = (r.Position - golPos).Magnitude
				if d < menor then
					menor = d
					melhor = r
				end
			end
		end
	end

	if playersFolder then
		for _, c in ipairs(playersFolder:GetChildren()) do
			avaliar(c)
		end
	end
	if npcsFolder then
		for _, c in ipairs(npcsFolder:GetChildren()) do
			avaliar(c)
		end
	end

	if melhor and menor <= GK_MAX_DIST_GOL then
		return melhor
	end
	return nil
end

local function calculaMira(raiz, bolaPos, ladoForcado)
	local gol = escolheGol(raiz)
	if not gol then
		return nil
	end
	local gcf, gsize = infoGol(gol)
	local golPos = gcf.Position

	local dirGol = Vector3.new(golPos.X - raiz.Position.X, 0, golPos.Z - raiz.Position.Z)
	if dirGol.Magnitude < 0.1 then
		return nil
	end
	local dirU = dirGol.Unit

	local ax = Vector3.new(gcf.RightVector.X, 0, gcf.RightVector.Z)
	local az = Vector3.new(gcf.LookVector.X, 0, gcf.LookVector.Z)
	local eixo, largura
	if ax.Magnitude < 0.01 or az.Magnitude < 0.01 then
		eixo = Vector3.new(-dirU.Z, 0, dirU.X)
	else
		ax = ax.Unit
		az = az.Unit
		if math.abs(ax:Dot(dirU)) <= math.abs(az:Dot(dirU)) then
			eixo = ax
			largura = gsize.X
		else
			eixo = az
			largura = gsize.Z
		end
	end

	local meia = CANTO_PADRAO
	if largura and largura >= 4 and largura <= 90 then
		meia = (largura / 2) * MARGEM_TRAVE
	end

	local gk = achaGoleiro(golPos)
	local gkLat = nil
	if gk then
		gkLat = (gk.Position - golPos):Dot(eixo)
		local distChute = Vector3.new(golPos.X - bolaPos.X, 0, golPos.Z - bolaPos.Z).Magnitude
		local tChute = distChute / VEL_CHUTE_ESTIMADA
		gkLat += gk.AssemblyLinearVelocity:Dot(eixo) * tChute
	end

	local meuLat = (raiz.Position - golPos):Dot(eixo)
	local pesoLonge = gk and PREFERE_TRAVE_LONGE or 1

	local melhorLat, melhorScore = 0, -math.huge
	for _, f in ipairs({ -1, -0.6, 0.6, 1 }) do
		local sinalF = f >= 0 and 1 or -1
		if not ladoForcado or sinalF == ladoForcado then
			local lat = f * meia
			local score = (gkLat and math.abs(lat - gkLat) or 0) + pesoLonge * math.abs(lat - meuLat)
			if score > melhorScore then
				melhorScore = score
				melhorLat = lat
			end
		end
	end

	local sinal = melhorLat >= 0 and 1 or -1
	local latAlvo = melhorLat
	if CURVA_ATIVA then
		latAlvo = melhorLat + sinal * CURVA_DESVIO
	end

	local ponto = golPos + eixo * latAlvo
	return {
		ponto = Vector3.new(ponto.X, raiz.Position.Y, ponto.Z),
		nomeGol = gol.Parent and gol.Parent.Name or "?",
		temGk = gk ~= nil,
		lat = melhorLat,
		gkLat = gkLat,
		eixo = eixo,
		dirU = dirU,
	}
end

local mirando = false
local inicioMira = 0
local ultimoCalculo = 0
local pontoMira = nil
local ladoTravado = nil
local humanoidTravado = nil
local ultimaCurva = 0

local function pararMira()
	mirando = false
	pontoMira = nil
	ladoTravado = nil
	if humanoidTravado then
		if humanoidTravado.Parent then
			humanoidTravado.AutoRotate = true
		end
		humanoidTravado = nil
	end
end

local function recalcular(raiz)
	local bola = bolaMaisProxima(raiz.Position)
	local bolaPos = bola and bola.Position or raiz.Position
	local info = calculaMira(raiz, bolaPos, ladoTravado)
	if info then
		pontoMira = info.ponto
	end
	return info
end

local function iniciarMira()
	if not ativo.mira or fakeAtivo or modoGoleiro then
		return
	end
	local char = player.Character
	local humanoid = char and char:FindFirstChildOfClass("Humanoid")
	local raiz = char and char:FindFirstChild("HumanoidRootPart")
	if not (humanoid and raiz) or humanoid.Health <= 0 then
		return
	end

	local bola, dist = bolaMaisProxima(raiz.Position)
	if not bola or dist > ALCANCE_BOLA then
		log("Sem bola dentro da zona verde, não mirou")
		return
	end

	ladoTravado = nil
	local info = recalcular(raiz)
	if not info then
		log("Gol não encontrado")
		return
	end
	ladoTravado = info.lat >= 0 and 1 or -1

	mirando = true
	inicioMira = os.clock()
	ultimoCalculo = inicioMira

	local textoCurva = "sem curva"
	if CURVA_ATIVA and os.clock() - ultimaCurva >= CURVA_COOLDOWN then
		local direita = info.dirU:Cross(Vector3.yAxis)
		local curvaMundo = -info.eixo * ladoTravado
		local lado = curvaMundo:Dot(direita) >= 0 and 1 or -1
		if INVERTER_CURVA then
			lado = -lado
		end
		if deslizar(lado) then
			ultimaCurva = os.clock()
			textoCurva = lado > 0 and "direita" or "esquerda"
		else
			textoCurva = okVIM and "ocupado" or "sem executor"
		end
	end

	log(
		"Mirando no", info.nomeGol,
		"| goleiro:", info.temGk and "sim" or "não",
		"| canto:", math.floor(info.lat * 10) / 10,
		"| goleiro lat:", info.gkLat and math.floor(info.gkLat * 10) / 10 or "-",
		"| curva:", textoCurva
	)
end

local function ehToque(input)
	return input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1
end

local function observarBotao(botao, aoApertar)
	botao.InputBegan:Connect(function(input)
		if not ehToque(input) or ehVirtual(botao) then
			return
		end
		ultimoToqueUsuario = os.clock()
		if botao == botao2 then
			inicioHoldReal2 = os.clock()
		end
		if aoApertar then
			aoApertar()
		end
	end)
	botao.InputEnded:Connect(function(input)
		if not ehToque(input) or ehVirtual(botao) then
			return
		end
		ultimoToqueUsuario = os.clock()
		if botao == botao2 then
			inicioHoldReal2 = nil
		end
	end)
end

observarBotao(botao0)
observarBotao(botao2, iniciarMira)

RunService:BindToRenderStep("MiraChute", Enum.RenderPriority.Input.Value + 1, function()
	if not mirando then
		return
	end

	local agora = os.clock()
	local char = player.Character
	local humanoid = char and char:FindFirstChildOfClass("Humanoid")
	local raiz = char and char:FindFirstChild("HumanoidRootPart")

	if not (humanoid and raiz) or humanoid.Health <= 0 or agora > inicioMira + JANELA_MIRA then
		pararMira()
		return
	end

	if agora - ultimoCalculo >= RECALCULAR_A_CADA then
		ultimoCalculo = agora
		recalcular(raiz)
	end

	if not pontoMira then
		return
	end

	if humanoidTravado ~= humanoid then
		humanoid.AutoRotate = false
		humanoidTravado = humanoid
	end

	local plano = Vector3.new(pontoMira.X - raiz.Position.X, 0, pontoMira.Z - raiz.Position.Z)
	if plano.Magnitude > 0.1 then
		local alvoCF = CFrame.lookAt(raiz.Position, raiz.Position + plano)
		raiz.CFrame = raiz.CFrame:Lerp(alvoCF, FORCA_MIRA)
	end
end)

local pastaVisual = Instance.new("Folder")
pastaVisual.Name = "SistemaAtacanteVisuais"
pastaVisual.Parent = workspace

local function criarEsfera(nome, diametro, cor, transparencia)
	local p = Instance.new("Part")
	p.Name = nome
	p.Shape = Enum.PartType.Ball
	p.Size = Vector3.new(diametro, diametro, diametro)
	p.Anchored = true
	p.CanCollide = false
	p.CanQuery = false
	p.CanTouch = false
	p.CastShadow = false
	p.Material = Enum.Material.SmoothPlastic
	p.Color = cor
	p.Transparency = transparencia
	p.Parent = pastaVisual
	return p
end

local zonaGk = criarEsfera("ZonaGoleiro", GK_ZONA_DIAMETRO, Color3.fromRGB(255, 255, 255), 1)
local zonaCerteza = criarEsfera("ZonaCerteza", CERTEZA_DIAMETRO, Color3.fromRGB(100, 255, 150), 1)
local noBola = criarEsfera("BolaAzul", BOLA_DIAMETRO, Color3.fromRGB(0, 120, 255), 1)
local marcador = criarEsfera("AlvoMira", MARCADOR_DIAMETRO, Color3.fromRGB(255, 220, 60), 1)

local golVisual = nil
local gkVisual = nil
local ultimoGkScan = 0

local function atualizaGkVisual(raiz)
	local escolhido
	if FORCAR_GOL then
		escolhido = pegaGol(FORCAR_GOL)
	else
		local g1, g2 = pegaGol("Team1"), pegaGol("Team2")
		if g1 and g2 then
			local s1 = pontuacaoOlhar(raiz, g1)
			local s2 = pontuacaoOlhar(raiz, g2)
			if golVisual == g1 then
				escolhido = (s2 > s1 + 0.3) and g2 or g1
			elseif golVisual == g2 then
				escolhido = (s1 > s2 + 0.3) and g1 or g2
			else
				escolhido = s1 >= s2 and g1 or g2
			end
		else
			escolhido = g1 or g2
		end
	end
	golVisual = escolhido
	gkVisual = escolhido and achaGoleiro(infoGol(escolhido).Position) or nil
end

RunService:BindToRenderStep("SistemaAtacanteVisuais", Enum.RenderPriority.Input.Value + 1, function()
	local char = player.Character
	local humanoid = char and char:FindFirstChildOfClass("Humanoid")
	local raiz = char and char:FindFirstChild("HumanoidRootPart")

	if modoGoleiro or not (humanoid and raiz) or humanoid.Health <= 0 then
		zonaCerteza.Transparency = 1
		zonaGk.Transparency = 1
		noBola.Transparency = 1
		marcador.Transparency = 1
		return
	end

	zonaCerteza.CFrame = CFrame.new(raiz.Position)

	local agora = os.clock()
	if agora - ultimoGkScan > 0.3 then
		ultimoGkScan = agora
		atualizaGkVisual(raiz)
	end

	if gkVisual and gkVisual.Parent then
		zonaGk.Transparency = GK_ZONA_TRANSP
		zonaGk.CFrame = CFrame.new(gkVisual.Position)
	else
		zonaGk.Transparency = 1
	end

	local bola, dist = bolaMaisProxima(raiz.Position)
	if bola then
		noBola.Transparency = BOLA_TRANSP
		noBola.CFrame = CFrame.new(bola.Position)
		zonaCerteza.Transparency = dist <= ALCANCE_BOLA and CERTEZA_TRANSP_PRONTA or CERTEZA_TRANSP
	else
		noBola.Transparency = 1
		zonaCerteza.Transparency = CERTEZA_TRANSP
	end

	if mirando and pontoMira then
		marcador.Transparency = 0.2
		marcador.CFrame = CFrame.new(pontoMira)
	else
		marcador.Transparency = 1
	end
end)

local function mesmoTime(modelo, minha)
	if not (IGNORAR_MEU_TIME and minha) then
		return false
	end
	local h = achaHandle(modelo)
	return h ~= nil and h:IsA("BasePart") and corParecida(h.Color, minha)
end

local function adversarioMaisPerto(raiz)
	local meu = meuModelo()
	local minha = minhaCor()
	local melhor, menor = nil, math.huge

	local function avaliar(modelo)
		if modelo == meu or modelo.Name == NOME_JOGADOR then
			return
		end
		local r = modelo:FindFirstChild("HumanoidRootPart") or modelo.PrimaryPart
		if not r then
			return
		end
		if mesmoTime(modelo, minha) then
			return
		end
		local d = (r.Position - raiz.Position).Magnitude
		if d < menor then
			menor = d
			melhor = modelo
		end
	end

	if playersFolder then
		for _, c in ipairs(playersFolder:GetChildren()) do
			avaliar(c)
		end
	end
	if npcsFolder then
		for _, c in ipairs(npcsFolder:GetChildren()) do
			avaliar(c)
		end
	end
	return melhor, menor
end

local function posicaoDoSom(som)
	local p = som.Parent
	if p and p:IsA("BasePart") then
		return p.Position, p
	end
	if p and p:IsA("Attachment") then
		return p.WorldPosition, p.Parent
	end
	return nil, nil
end

local ultimoSom = 0
local ultimoDrible = 0

local function aoTocarSom(som)
	local agora = os.clock()
	if agora - ultimoSom < 0.05 then
		return
	end
	ultimoSom = agora

	if not ativo.drible or fakeAtivo or modoGoleiro then
		return
	end

	local char = player.Character
	local humanoid = char and char:FindFirstChildOfClass("Humanoid")
	local raiz = char and char:FindFirstChild("HumanoidRootPart")
	if not (humanoid and raiz) or humanoid.Health <= 0 then
		return
	end

	local minha = minhaCor()
	local pos, parte = posicaoDoSom(som)
	local dist, quem, fonte

	if pos then
		if som:IsDescendantOf(char) then
			return
		end
		local modelo = parte and parte:FindFirstAncestorOfClass("Mo
    	if pos then
		if som:IsDescendantOf(char) then
			return
		end
		local modelo = parte and parte:FindFirstAncestorOfClass("Model")
		if modelo and mesmoTime(modelo, minha) then
			return
		end
		dist = (pos - raiz.Position).Magnitude
		quem = modelo and modelo.Name or "?"
		fonte = "posição do som"
	else
		local modelo, d = adversarioMaisPerto(raiz)
		if not modelo then
			return
		end
		dist = d
		quem = modelo.Name
		fonte = "adversário mais perto"
	end

	local resumo = string.format("%s | %s a %.1f studs (%s)", som.Name, quem, dist, fonte)

	if dist > DISTANCIA_TACKLE then
		return
	end

	if SO_COM_BOLA then
		local _, db = bolaMaisProxima(raiz.Position)
		if not db or db > DISTANCIA_BOLA then
			return
		end
	end

	if agora - ultimoDrible < (DRIBLE_COOLDOWN * 0.5) then
		return
	end
	ultimoDrible = agora

	task.delay(ATRASO_CLIQUE, function()
		apertarDribbleMultiplo()
		log("DRIBLE AUTOMÁTICO DISPARADO!", resumo)
	end)
end

local ligados = {}

local function ligarSom(obj)
	if ligados[obj] or not obj:IsA("Sound") then
		return
	end
	if string.sub(obj.Name, 1, #PREFIXO_SOM) ~= PREFIXO_SOM then
		return
	end
	ligados[obj] = true

	obj.Played:Connect(function()
		aoTocarSom(obj)
	end)
	obj:GetPropertyChangedSignal("Playing"):Connect(function()
		if obj.Playing then
			aoTocarSom(obj)
		end
	end)
	obj.Destroying:Connect(function()
		ligados[obj] = nil
		if obj.PlayOnRemove then
			aoTocarSom(obj)
		end
	end)
	if obj.IsPlaying then
		task.defer(aoTocarSom, obj)
	end
end

for _, o in ipairs(SoundService:GetDescendants()) do
	ligarSom(o)
end
SoundService.DescendantAdded:Connect(ligarSom)

for _, o in ipairs(workspace:GetDescendants()) do
	if o.Name == NOME_BOLA then
		registrarBola(o)
	end
	ligarSom(o)
end

workspace.DescendantAdded:Connect(function(o)
	if o.Name == NOME_BOLA then
		task.defer(registrarBola, o)
	end
	if o:IsA("Sound") then
		task.defer(ligarSom, o)
	end
end)

local function avaliaPressao(raiz, dirGol, minha)
	local meu = meuModelo()
	local score, perto, maisPerto = 0, 0, math.huge

	local function avaliar(modelo)
		if modelo == meu or modelo.Name == NOME_JOGADOR then
			return
		end
		local r = modelo:FindFirstChild("HumanoidRootPart") or modelo.PrimaryPart
		if not r then
			return
		end

		local h = achaHandle(modelo)
		if h and h:IsA("BasePart") then
			local cor = h.Color
			if corParecida(cor, minha) then
				return
			end
			if not CONTAR_GOLEIRO and ehBranca(cor) then
				return
			end
		elseif not CONTAR_SEM_CAMISA then
			return
		end

		local rel = r.Position - raiz.Position
		local relH = Vector3.new(rel.X, 0, rel.Z)
		local d = math.max(relH.Magnitude, 0.01)
		if d > RAIO_PRESSAO then
			return
		end

		local w = (1 - d / RAIO_PRESSAO) ^ 2

		local vel = r.AssemblyLinearVelocity
		local vH = Vector3.new(vel.X, 0, vel.Z)
		local paraMim = -relH / d
		local fechando = math.clamp(vH:Dot(paraMim) / 16, 0, 1)
		w *= 1 + 0.5 * fechando

		if dirGol then
			local frente = math.clamp((relH / d):Dot(dirGol), 0, 1)
			w *= 0.7 + 0.3 * frente
		end

		score += w
		if d <= RAIO_PRESSAO / 2 then
			perto += 1
		end
		if d < maisPerto then
			maisPerto = d
		end
	end

	if playersFolder then
		for _, c in ipairs(playersFolder:GetChildren()) do
			avaliar(c)
		end
	end
	if npcsFolder then
		for _, c in ipairs(npcsFolder:GetChildren()) do
			avaliar(c)
		end
	end

	return score, perto, maisPerto
end

local ultimoFake = 0

local function fakeShot(resumo)
	if fakeAtivo or gestoAtivo then
		return
	end
	if not okVIM then
		logLento("semvim", "VirtualInputManager indisponível: o fake shot só funciona no executor")
		return
	end

	fakeAtivo = true
	gestoAtivo = true
	ultimoFake = os.clock()

	local idChute = novoId()
	marcarVirtual(botao2, FAKE_SEGURAR + 2 * ANTECEDENCIA + DURACAO_TOQUE + SOLTAR_CHUTE_DEPOIS + 0.4)

	log("FAKE SHOT!", resumo)

	task.spawn(function()
		local ok, err = pcall(function()
			enviarToque(idChute, 0, botao2)
			task.wait(ANTECEDENCIA)
			task.wait(FAKE_SEGURAR)
			task.wait(ANTECEDENCIA)
			tocarBotao(botao0)
			task.wait(SOLTAR_CHUTE_DEPOIS)
		end)
		enviarToque(idChute, 2, botao2)
		if not ok then
			log("Erro no fake shot:", err)
		end
		task.delay(0.3, function()
			fakeAtivo = false
			gestoAtivo = false
		end)
	end)
end

local acumulador = 0
local confirmacoes = 0

RunService.Heartbeat:Connect(function(dt)
	acumulador += dt
	if acumulador < INTERVALO_ANALISE then
		return
	end
	acumulador = 0

	if not ativo.fake or fakeAtivo or gestoAtivo or modoGoleiro then
		confirmacoes = 0
		return
	end

	local char = player.Character
	local humanoid = char and char:FindFirstChildOfClass("Humanoid")
	local raiz = char and char:FindFirstChild("HumanoidRootPart")
	if not (humanoid and raiz) or humanoid.Health <= 0 then
		confirmacoes = 0
		return
	end

	local _, db = bolaMaisProxima(raiz.Position)
	if not db or db > BOLA_PERTO then
		confirmacoes = 0
		return
	end

	local minha = minhaCor()
	if not minha then
		logLento("semcor", "Camisa não encontrada: não consigo diferenciar os times")
		confirmacoes = 0
		return
	end

	local dirGol = nil
	local gol = escolheGol(raiz)
	if gol then
		local gp = infoGol(gol).Position
		local dv = Vector3.new(gp.X - raiz.Position.X, 0, gp.Z - raiz.Position.Z)
		if dv.Magnitude > 0.1 then
			dirGol = dv.Unit
		end
		if SO_NO_ATAQUE and dv.Magnitude > DIST_GOL_MAX then
			logLento("longegol", "Longe do gol (" .. math.floor(dv.Magnitude) .. " studs), sem fake shot")
			confirmacoes = 0
			return
		end
	end

	local pressao, perto, mais = avaliaPressao(raiz, dirGol, minha)
	local resumo = string.format("pressão %.2f | adversários perto %d | mais perto %.1f", pressao, perto, mais == math.huge and 0 or mais)

	if pressao > PRESSAO_MIN * 0.5 then
		logLento("pressao", resumo)
	end

	if pressao >= PRESSAO_MIN then
		confirmacoes += 1
	else
		confirmacoes = 0
	end

	if confirmacoes >= CONFIRMAR then
		if usuarioOcupado() then
			logLento("ocupado", "Você está nos botões, fake shot pausado")
			return
		end
		if os.clock() - ultimoFake < FAKE_COOLDOWN then
			return
		end
		confirmacoes = 0
		fakeShot(resumo)
	end
end)

log("Sistema iniciado | executor:", okVIM and "sim" or "não")
