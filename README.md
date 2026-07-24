local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")
local CollectionService = game:GetService("CollectionService")
local Players = game:GetService("Players")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- ============================================================
-- ESTADO
-- ============================================================
local nivelAtual = 1

local cacheEfeitos = setmetatable({}, {__mode = "k"})
local cacheLuzes = setmetatable({}, {__mode = "k"})
local cacheMeshParts = setmetatable({}, {__mode = "k"})
local cachePostEffects = setmetatable({}, {__mode = "k"})
local atmosferaGuardada = {}

local botoesNivel = {}
local botaoAbrir

local original = {
	tecnologia = Lighting.Technology,
	sombras = Lighting.GlobalShadows,
	diffuse = Lighting.EnvironmentDiffuseScale,
	specular = Lighting.EnvironmentSpecularScale,
	terrenoDecoracao = Workspace.Terrain.Decoration,
}

-- ============================================================
-- FUNÇÕES DE OTIMIZAÇÃO
-- ============================================================
local function configurarObjeto(obj)
	if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam")
		or obj:IsA("Fire") or obj:IsA("Smoke") then
		if cacheEfeitos[obj] == nil then
			cacheEfeitos[obj] = obj.Enabled
		end
		obj.Enabled = (nivelAtual < 3) and cacheEfeitos[obj] or false

	elseif obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then
		if cacheLuzes[obj] == nil then
			cacheLuzes[obj] = obj.Shadows
		end
		obj.Shadows = (nivelAtual < 3) and cacheLuzes[obj] or false

	elseif obj:IsA("MeshPart") then
		if cacheMeshParts[obj] == nil then
			cacheMeshParts[obj] = obj.RenderFidelity
		end
		obj.RenderFidelity = (nivelAtual < 4) and cacheMeshParts[obj] or Enum.RenderFidelity.Performance
	end
end

local function configurarAtmosfera()
	if nivelAtual < 3 then
		for obj, pai in pairs(atmosferaGuardada) do
			obj.Parent = pai
			atmosferaGuardada[obj] = nil
		end
	else
		for _, obj in ipairs(Workspace:GetDescendants()) do
			if (obj:IsA("Atmosphere") or obj:IsA("Clouds")) and obj.Parent then
				atmosferaGuardada[obj] = obj.Parent
				obj.Parent = nil
			end
		end
	end
end

local function esconderObjetoTag(obj)
	local esconder = nivelAtual >= 5
	local partes = obj:IsA("BasePart") and {obj} or obj:GetDescendants()
	for _, parte in ipairs(partes) do
		if parte:IsA("BasePart") then
			parte.LocalTransparencyModifier = esconder and 1 or 0
		end
	end
end

local function aplicarTagOculta(tag)
	for _, obj in ipairs(CollectionService:GetTagged(tag)) do
		esconderObjetoTag(obj)
	end
end

local function configurarPersonagem(personagem)
	local esconder = nivelAtual >= 5
	for _, parte in ipairs(personagem:GetDescendants()) do
		if parte:IsA("BasePart") or parte:IsA("Decal") then
			parte.LocalTransparencyModifier = esconder and 1 or 0
		end
	end
end

local function conectarJogador(jogador)
	if jogador == LocalPlayer then return end
	jogador.CharacterAdded:Connect(configurarPersonagem)
	if jogador.Character then
		configurarPersonagem(jogador.Character)
	end
end

local function atualizarTodosPersonagens()
	for _, jogador in ipairs(Players:GetPlayers()) do
		if jogador ~= LocalPlayer and jogador.Character then
			configurarPersonagem(jogador.Character)
		end
	end
end

local function atualizarMenuVisual()
	for i, botao in ipairs(botoesNivel) do
		botao.BackgroundColor3 = (i == nivelAtual) and Color3.fromRGB(70, 130, 220) or Color3.fromRGB(45, 45, 52)
	end
	if botaoAbrir then
		botaoAbrir.Text = "N" .. nivelAtual
	end
end

-- ============================================================
-- APLICAR NÍVEL (função central chamada pelos botões do menu)
-- ============================================================
local function aplicarNivel(nivel)
	nivelAtual = math.clamp(nivel, 1, 5)

	if nivelAtual >= 2 then
		Lighting.GlobalShadows = false
		Lighting.Technology = Enum.Technology.Compatibility
		Lighting.EnvironmentDiffuseScale = 0
		Lighting.EnvironmentSpecularScale = 0
	else
		Lighting.GlobalShadows = original.sombras
		Lighting.Technology = original.tecnologia
		Lighting.EnvironmentDiffuseScale = original.diffuse
		Lighting.EnvironmentSpecularScale = original.specular
	end

	for _, effect in ipairs(Lighting:GetChildren()) do
		if effect:IsA("PostEffect") then
			if cachePostEffects[effect] == nil then
				cachePostEffects[effect] = effect.Enabled
			end
			effect.Enabled = (nivelAtual < 2) and cachePostEffects[effect] or false
		end
	end

	configurarAtmosfera()
	Workspace.Terrain.Decoration = (nivelAtual < 3) and original.terrenoDecoracao or false

	for _, obj in ipairs(Workspace:GetDescendants()) do
		configurarObjeto(obj)
	end

	aplicarTagOculta("Decoracao")
	aplicarTagOculta("Inimigo")
	atualizarTodosPersonagens()

	atualizarMenuVisual()
	print("[Otimização] Nível " .. nivelAtual .. " aplicado.")
end

-- ============================================================
-- GUI
-- ============================================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "MenuOtimizacaoFPS"
screenGui.ResetOnSpawn = false
screenGui.Parent = PlayerGui

botaoAbrir = Instance.new("TextButton")
botaoAbrir.Name = "BotaoAbrir"
botaoAbrir.Size = UDim2.new(0, 46, 0, 46)
botaoAbrir.Position = UDim2.new(0, 10, 0, 10)
botaoAbrir.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
botaoAbrir.TextColor3 = Color3.fromRGB(255, 255, 255)
botaoAbrir.Font = Enum.Font.GothamBold
botaoAbrir.TextSize = 16
botaoAbrir.Text = "N1"
botaoAbrir.Parent = screenGui

local cornerAbrir = Instance.new("UICorner")
cornerAbrir.CornerRadius = UDim.new(1, 0)
cornerAbrir.Parent = botaoAbrir

local frame = Instance.new("Frame")
frame.Name = "Menu"
frame.Size = UDim2.new(0, 220, 0, 0)
frame.AutomaticSize = Enum.AutomaticSize.Y
frame.Position = UDim2.new(0, 10, 0, 64)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
frame.Visible = false
frame.Parent = screenGui

local cornerFrame = Instance.new("UICorner")
cornerFrame.CornerRadius = UDim.new(0, 12)
cornerFrame.Parent = frame

local padding = Instance.new("UIPadding")
padding.PaddingTop = UDim.new(0, 12)
padding.PaddingBottom = UDim.new(0, 12)
padding.PaddingLeft = UDim.new(0, 12)
padding.PaddingRight = UDim.new(0, 12)
padding.Parent = frame

local lista = Instance.new("UIListLayout")
lista.Padding = UDim.new(0, 8)
lista.SortOrder = Enum.SortOrder.LayoutOrder
lista.Parent = frame

local titulo = Instance.new("TextLabel")
titulo.Size = UDim2.new(1, 0, 0, 28)
titulo.BackgroundTransparency = 1
titulo.Text = "Otimização de FPS"
titulo.TextColor3 = Color3.fromRGB(255, 255, 255)
titulo.Font = Enum.Font.GothamBold
titulo.TextSize = 15
titulo.LayoutOrder = 0
titulo.Parent = frame

local nomesNiveis = {
	"Nível 1 - Máxima Qualidade",
	"Nível 2 - Leve",
	"Nível 3 - Média",
	"Nível 4 - Pesada",
	"Nível 5 - Extrema",
}

for i, nome in ipairs(nomesNiveis) do
	local botao = Instance.new("TextButton")
	botao.Name = "Nivel" .. i
	botao.Size = UDim2.new(1, 0, 0, 34)
	botao.BackgroundColor3 = Color3.fromRGB(45, 45, 52)
	botao.Text = nome
	botao.TextColor3 = Color3.fromRGB(230, 230, 230)
	botao.Font = Enum.Font.Gotham
	botao.TextSize = 13
	botao.LayoutOrder = i
	botao.Parent = frame

	local cornerBotao = Instance.new("UICorner")
	cornerBotao.CornerRadius = UDim.new(0, 8)
	cornerBotao.Parent = botao

	botao.MouseButton1Click:Connect(function()
		aplicarNivel(i)
	end)

	botoesNivel[i] = botao
end

botaoAbrir.MouseButton1Click:Connect(function()
	frame.Visible = not frame.Visible
end)

-- ============================================================
-- EVENTOS E INICIALIZAÇÃO
-- ============================================================
for _, jogador in ipairs(Players:GetPlayers()) do
	conectarJogador(jogador)
end
Players.PlayerAdded:Connect(conectarJogador)

Workspace.DescendantAdded:Connect(configurarObjeto)

CollectionService:GetInstanceAddedSignal("Decoracao"):Connect(esconderObjetoTag)
CollectionService:GetInstanceAddedSignal("Inimigo"):Connect(esconderObjetoTag)

aplicarNivel(1)
