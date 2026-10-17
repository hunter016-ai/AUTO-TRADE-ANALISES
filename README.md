# AUTO-TRADE-ANALISES

//@version=6
indicator("AUTO TRADE 1.0", overlay=true, format=format.price)

// --- 1. SETTINGS ---
factor = input.float(3.0, "Sensibilité Supertrend", step=0.1, group="1. Tendance") 
atrPeriod = input.int(12, "Période Supertrend", group="1. Tendance")
emaLength = input.int(200, "EMA de Fond", group="1. Tendance")

adxThreshold = input.int(18, "Seuil ADX", minval=1, group="2. Filtres") 
useRSI = input.bool(true, "Filtre RSI (>50/<50)", group="2. Filtres")

useFibTarget = input.bool(false, "Utiliser TP Fib (1.618)", group="3. Exécution")
tp_ratio_manual = input.float(1.5, "Ratio R:R Manuel", step=0.1, group="3. Exécution")
sl_padding = input.float(1.5, "Marge SL (Multiplier ATR - Anti-Mèche)", step=0.1, group="3. Exécution")

// Options de Visuels
showDashboard = input.bool(true, "Afficher le Tableau de Bord", group="4. Visuel")
textSizeInput = input.string("Petit", "Taille du Tableau", options=["Ultra-Petit", "Petit", "Normal"], group="4. Visuel")

actual_ratio = useFibTarget ? 1.618 : tp_ratio_manual

// Attribution correcte des tailles v6
string dashboardTextSize = size.small
if textSizeInput == "Ultra-Petit"
    dashboardTextSize := size.small
else if textSizeInput == "Normal"
    dashboardTextSize := size.normal

// --- 2. CALCULS ---
[supertrend, direction] = ta.supertrend(factor, atrPeriod)
emaFilter = ta.ema(close, emaLength)
atrValue = ta.atr(14)
rsiValue = ta.rsi(close, 14)
[diplus, minus, adxValue] = ta.dmi(14, 14)

// Signaux au moment T
rawLong = ta.crossover(close, supertrend) and close > emaFilter
rawShort = ta.crossunder(close, supertrend) and close < emaFilter

rsiOkLong = useRSI ? (rsiValue > 50) : true
rsiOkShort = useRSI ? (rsiValue < 50) : true
adxOk = adxValue > adxThreshold 

// --- 3. LOGIQUE DE SIGNAL ---
goLong = rawLong[1] and rsiOkLong[1] and adxOk[1]
goShort = rawShort[1] and rsiOkShort[1] and adxOk[1]

// --- 4. COULEURS ---
flashyGreen = #00FF00
flashyPink = #FF00FF
barcolor(goLong ? flashyGreen : goShort ? flashyPink : na, title="Bougie Signal")

// --- 5. GESTION DES ENTRÉES ET DU RISQUE ---
var float lastEntry = na
var float lastSL = na
var float lastTP = na
var float lastRSI = na  
var int lastTime = na
var string lastType = "ATTENTE"

if goLong
    lastEntry := open 
    float lowestLow = math.min(low[1], math.min(low[2], low[3]))
    lastSL := lowestLow - (atrValue[1] * sl_padding) 
    float risk = lastEntry - lastSL
    lastTP := lastEntry + (risk * actual_ratio)
    lastTime := time
    lastType := "LONG"
    lastRSI := rsiValue[1] 

if goShort
    lastEntry := open 
    float highestHigh = math.max(high[1], math.max(high[2], high[3]))
    lastSL := highestHigh + (atrValue[1] * sl_padding) 
    float risk = lastSL - lastEntry
    lastTP := lastEntry - (risk * actual_ratio)
    lastTime := time
    lastType := "SHORT"
    lastRSI := rsiValue[1] 

// --- 6. AFFICHAGE INDICATEURS ---
plot(emaFilter, "EMA 200", color=color.white, linewidth=2)
plot(direction < 0 ? supertrend : na, "ST Vert", color=color.new(color.green, 50), style=plot.style_linebr)
plot(direction > 0 ? supertrend : na, "ST Rouge", color=color.new(color.red, 50), style=plot.style_linebr)

plotshape(goLong, title="Achat Flash", style=shape.labelup, location=location.belowbar, color=flashyGreen, size=size.small, text="GO", textcolor=color.black)
plotshape(goShort, title="Vente Flash", style=shape.labeldown, location=location.abovebar, color=flashyPink, size=size.small, text="GO", textcolor=color.white)

// --- 7. LIGNES SL/TP ---
var line slLine = na
var line tpLine = na
var label slLabel = na 
var label tpLabel = na 

if goLong or goShort
    line.delete(slLine)
    line.delete(tpLine)
    label.delete(slLabel)
    label.delete(tpLabel)
    
    slLine := line.new(bar_index, lastSL, bar_index + 15, lastSL, color=color.red, width=2)
    tpLine := line.new(bar_index, lastTP, bar_index + 15, lastTP, color=color.blue, width=2)
    
    string tpText = "OBJECTIF " + (useFibTarget ? "(Fib 1.618)" : "(Ratio " + str.tostring(actual_ratio, "#.#") + ")")
    tpLabel := label.new(x=bar_index + 5, y=lastTP, text=tpText + "\n" + str.tostring(lastTP,  "#"), color=color.blue, style=label.style_label_down, textcolor=color.white, size=size.small)
    slLabel := label.new(x=bar_index + 5, y=lastSL, text="STOP LOSS\n" + str.tostring(lastSL, "#"), color=color.red, style=label.style_label_up, textcolor=color.white, size=size.small)

// --- 8. TABLEAU DE BORD OPTIMISÉ (RÉDUIT) ---
var table dashboard = table.new(position.bottom_right, 2, 8, bgcolor=color.new(color.black, 40), border_width=1)

if barstate.islast and showDashboard and lastType != "ATTENTE"
    color typeCol = lastType == "LONG" ? flashyGreen : (lastType == "SHORT" ? flashyPink : color.gray)
    
    table.cell(dashboard, 0, 0, "SIGNAL", text_color=color.white, text_size=dashboardTextSize)
    table.cell(dashboard, 1, 0, str.format("{0,date,HH:mm}", lastTime), text_color=color.yellow, text_size=dashboardTextSize)
    table.cell(dashboard, 0, 1, "TYPE", text_color=color.white, text_size=dashboardTextSize)
    table.cell(dashboard, 1, 1, lastType, text_color=typeCol, text_size=dashboardTextSize)
    table.cell(dashboard, 0, 2, "RSI", text_color=color.silver, text_size=dashboardTextSize)
    color rsiCol = lastRSI >= 50 ? color.lime : color.red
    table.cell(dashboard, 1, 2, str.tostring(lastRSI, "#.1"), text_color=rsiCol, text_size=dashboardTextSize)
    table.cell(dashboard, 0, 3, "ADX", text_color=color.silver, text_size=dashboardTextSize)
    table.cell(dashboard, 1, 3, str.tostring(adxValue, "#.1"), text_color=adxValue > adxThreshold ? color.lime : color.red, text_size=dashboardTextSize)
    table.cell(dashboard, 0, 4, "R:R", text_color=color.white, text_size=dashboardTextSize)
    table.cell(dashboard, 1, 4, str.tostring(actual_ratio, "#.1"), text_color=#FFD700, text_size=dashboardTextSize)
    table.cell(dashboard, 0, 5, "PRIX", text_color=color.white, text_size=dashboardTextSize)
    table.cell(dashboard, 1, 5, str.tostring(lastEntry, "#"), text_color=color.white, text_size=dashboardTextSize)
    table.cell(dashboard, 0, 6, "TP", text_color=color.blue, text_size=dashboardTextSize)
    table.cell(dashboard, 1, 6, str.tostring(lastTP, "#"), text_color=color.blue, text_size=dashboardTextSize)
    table.cell(dashboard, 0, 7, "SL", text_color=color.red, text_size=dashboardTextSize)
    table.cell(dashboard, 1, 7, str.tostring(lastSL, "#"), text_color=color.red, text_size=dashboardTextSize)

// --- 9. ALERTES AUTOMATIQUES ---
if goLong
    alert("🟢 GO LONG - Entrée: " + str.tostring(open, "#") + " | SL: " + str.tostring(lastSL, "#") + " | TP: " + str.tostring(lastTP, "#"), alert.freq_once_per_bar_close)

if goShort
    alert("🔴 GO SHORT - Entrée: " + str.tostring(open, "#") + " | SL: " + str.tostring(lastSL, "#") + " | TP: " + str.tostring(lastTP, "#"), alert.freq_once_per_bar_close) 


