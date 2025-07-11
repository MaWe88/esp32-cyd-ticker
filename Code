#ifndef LV_USE_GESTURE
#define LV_USE_GESTURE 1
#endif
#include <FS.h>
using fs::FS;
#include <lvgl.h>
#include <TFT_eSPI.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <ArduinoJson.h>
#include "backgroundstonk.h"
#include <WiFiManager.h>
#include <XPT2046_Touchscreen.h>
#include <vector>
#include <Preferences.h>
Preferences prefs;

#define XPT2046_IRQ 36
#define XPT2046_MOSI 32
#define XPT2046_MISO 39
#define XPT2046_CLK 25
#define XPT2046_CS 33

SPIClass touchscreenSPI(VSPI);
XPT2046_Touchscreen touchscreen(XPT2046_CS, XPT2046_IRQ);

static lv_indev_drv_t indev_drv;

#define STATUS_LED_RED_PIN 4
#define STATUS_LED_GREEN_PIN 16
#define STATUS_LED_BLUE_PIN 17


String finnhub_api_key_config = "";
String coingecko_api_key_config = "";

bool auto_interval_enabled = true;  
unsigned long switchInterval = 60000;

bool status_led_enabled = true;


struct SymbolInfo {
  String displayName;
  String symbol;
  String apiProvider;
  bool isCrypto;
  String originalInput;  
  String coinGeckoId;    
};
unsigned long touchPressStart = 0;
bool touchPressed = false;
bool resetInProgress = false;

std::vector<SymbolInfo> symbols;
int numSymbols = 0;
const int MAX_SYMBOLS = 15;

int currentSymbolIndex = 0;

unsigned long lastSwitchTime = 0;
unsigned long lastApiCall = 0;


static const uint16_t screenWidth = 320;
static const uint16_t screenHeight = 240;
static lv_disp_draw_buf_t draw_buf;
static lv_color_t* buf;
TFT_eSPI tft = TFT_eSPI();
lv_disp_t* disp;


lv_obj_t* background_img;
lv_obj_t* symbol_panel;
lv_obj_t* price_panel;
lv_obj_t* percent_panel;
lv_obj_t* label_symbol;
lv_obj_t* label_price;
lv_obj_t* label_percent;
lv_obj_t* validation_overlay = nullptr;
lv_obj_t* validation_label = nullptr;

struct MarketData {
  String displayName;
  String symbol;
  float currentPrice;
  float change;
  float changePercent;
  bool dataValid;
  bool isCrypto;
  String apiProvider;


  float lastValidPrice;
  float lastValidChange;
  float lastValidChangePercent;
  bool hasBackupData;
};


String extractJSON(const String& response);
bool fetchCoinGeckoData(MarketData& data);
bool fetchFinnhubData(const char* symbol, MarketData& data);
bool fetchMarketData(int index, MarketData& data);
void updateDisplay(const MarketData& data);
void displaySetupInstructions(const char* ssid);
void updateStatusLED(bool wifiConnected);
void showValidationMessage(const String& message);
void hideValidationMessage();
void longPressResetHandler(lv_event_t* e);
String tickerToCoinGeckoId(const String& ticker);
String getCompanyName(const String& symbol);
String getCoinName(const String& coinId);
bool validateSymbol(const String& symbol, bool isCrypto);

MarketData* marketData = nullptr;

WiFiClientSecure client;

bool validateSymbol(const String& symbol, bool isCrypto) {
  if (symbol.length() == 0) return false;

  if (isCrypto) {
    
    String coinGeckoId = tickerToCoinGeckoId(symbol);

    
    String url = "/api/v3/simple/price?ids=" + coinGeckoId + "&vs_currencies=usd";
    if (!client.connect("api.coingecko.com", 443)) {
      Serial.println("Failed to connect to CoinGecko for validation");
      return false;
    }
Serial.println("Sending request with API Key: " + coingecko_api_key_config);

    client.print(String("GET ") + url + " HTTP/1.0\r\n" +
           "Host: api.coingecko.com\r\n" +
           "User-Agent: ESP32-CYD-Ticker\r\n" +
           "x-cg-demo-api-key: " + coingecko_api_key_config + "\r\n" +
           "Connection: close\r\n\r\n");

    unsigned long timeout = millis() + 5000;
    while (!client.available() && millis() < timeout) {
      delay(10);
    }

    if (!client.available()) {
      client.stop();
      return false;
    }

    
    while (client.available()) {
      String line = client.readStringUntil('\n');
      if (line == "\r") break;
    }

    String response = "";
    while (client.available()) {
      response += client.readString();
    }
    client.stop();

    Serial.println("CoinGecko Validation Response for " + symbol + " (" + coinGeckoId + "):");
    Serial.println(response);

    
    if (response.indexOf("429") >= 0 || response.indexOf("Rate Limit") >= 0) {
      Serial.println("Rate limit exceeded, treating as valid symbol: " + symbol);
      return true;  
    }

    
    return (response.indexOf(coinGeckoId) >= 0 && response.indexOf("usd") >= 0);

  } else {
    
    if (finnhub_api_key_config.length() == 0) return false;

    String url = "/api/v1/quote?symbol=" + symbol + "&token=" + finnhub_api_key_config;
    if (!client.connect("finnhub.io", 443)) {
      return false;
    }

    client.print(String("GET ") + url + " HTTP/1.1\r\n" + "Host: finnhub.io\r\n" + "Connection: close\r\n\r\n");

    unsigned long timeout = millis() + 5000;
    while (!client.available() && millis() < timeout) {
      delay(10);
    }

    if (!client.available()) {
      client.stop();
      return false;
    }

    
    while (client.available()) {
      String line = client.readStringUntil('\n');
      if (line == "\r") break;
    }

    String response = "";
    while (client.available()) {
      response += client.readString();
    }
    client.stop();

    
    DynamicJsonDocument doc(1024);
    DeserializationError error = deserializeJson(doc, response);
    if (error) return false;

    
    return (doc.containsKey("c") && doc["c"].as<float>() > 0);
  }
}



String tickerToCoinGeckoId(const String& ticker) {
  String upperTicker = ticker;
  upperTicker.toUpperCase();

  // Top 50 Cryptocurrencies
  if (upperTicker == "BTC") return "bitcoin";
  if (upperTicker == "ETH") return "ethereum";
  if (upperTicker == "USDT") return "tether";
  if (upperTicker == "BNB") return "binancecoin";
  if (upperTicker == "SOL") return "solana";
  if (upperTicker == "USDC") return "usd-coin";
  if (upperTicker == "XRP") return "ripple";
  if (upperTicker == "STETH") return "staked-ether";
  if (upperTicker == "DOGE") return "dogecoin";
  if (upperTicker == "TRX") return "tron";
  if (upperTicker == "TON") return "the-open-network";
  if (upperTicker == "ADA") return "cardano";
  if (upperTicker == "WBTC") return "wrapped-bitcoin";
  if (upperTicker == "AVAX") return "avalanche-2";
  if (upperTicker == "SHIB") return "shiba-inu";
  if (upperTicker == "LINK") return "chainlink";
  if (upperTicker == "BCH") return "bitcoin-cash";
  if (upperTicker == "DOT") return "polkadot";
  if (upperTicker == "LEO") return "leo-token";
  if (upperTicker == "LTC") return "litecoin";
  if (upperTicker == "NEAR") return "near";
  if (upperTicker == "MATIC") return "polygon";
  if (upperTicker == "PEPE") return "pepe";
  if (upperTicker == "DAI") return "dai";
  if (upperTicker == "ICP") return "internet-computer";
  if (upperTicker == "ETC") return "ethereum-classic";
  if (upperTicker == "HBAR") return "hedera-hashgraph";
  if (upperTicker == "APT") return "aptos";
  if (upperTicker == "UNI") return "uniswap";
  if (upperTicker == "FDUSD") return "first-digital-usd";
  if (upperTicker == "CRO") return "crypto-com-chain";
  if (upperTicker == "RENDER") return "render-token";  
  if (upperTicker == "XLM") return "stellar";
  if (upperTicker == "ATOM") return "cosmos";
  if (upperTicker == "FIL") return "filecoin";
  if (upperTicker == "MNT") return "mantle";
  if (upperTicker == "OKB") return "okb";
  if (upperTicker == "HYPE") return "hyperliquid";
  if (upperTicker == "ARB") return "arbitrum";
  if (upperTicker == "XMR") return "monero";
  if (upperTicker == "BONK") return "bonk";
  if (upperTicker == "FET") return "fetch-ai";
  if (upperTicker == "OP") return "optimism";
  if (upperTicker == "CAKE") return "pancakeswap-token";
  if (upperTicker == "LIDO") return "lido-dao";
  if (upperTicker == "USDE") return "ethena-usde";
  if (upperTicker == "BRETT") return "brett";
  if (upperTicker == "AAVE") return "aave";
  if (upperTicker == "SUI") return "sui";

  // Top 51-100 Cryptocurrencies
  if (upperTicker == "CRV") return "curve-dao-token";
  if (upperTicker == "PENDLE") return "pendle";
  if (upperTicker == "GRASS") return "grass";
  if (upperTicker == "ALGO") return "algorand";
  if (upperTicker == "FLUX") return "flux";
  if (upperTicker == "EIGEN") return "eigenlayer";
  if (upperTicker == "DYDX") return "dydx";
  if (upperTicker == "COMP") return "compound";
  if (upperTicker == "SUSHI") return "sushi";
  if (upperTicker == "USDD") return "usdd";
  if (upperTicker == "FLOKI") return "floki";
  if (upperTicker == "THETA") return "theta-token";
  if (upperTicker == "TUSD") return "true-usd";
  if (upperTicker == "ETHFI") return "ether-fi";
  if (upperTicker == "LUNC") return "terra-luna";
  if (upperTicker == "SAND") return "the-sandbox";
  if (upperTicker == "MANA") return "decentraland";
  if (upperTicker == "RUNE") return "thorchain";
  if (upperTicker == "METH") return "mantle-staked-ether";
  if (upperTicker == "GALA") return "gala";
  if (upperTicker == "USUAL") return "usual";
  if (upperTicker == "PNUT") return "peanut-the-squirrel";
  if (upperTicker == "INJ") return "injective-protocol";
  if (upperTicker == "MOVE") return "move";
  if (upperTicker == "FLOW") return "flow";
  if (upperTicker == "IOTA") return "iota";
  if (upperTicker == "VET") return "vechain";
  if (upperTicker == "VEN") return "vechain";  
  if (upperTicker == "ORDI") return "ordinals";
  if (upperTicker == "MSOL") return "marinade-staked-sol";
  if (upperTicker == "USTC") return "terrausd";
  if (upperTicker == "EZETH") return "renzo-restaked-eth";
  if (upperTicker == "RETH") return "rocket-pool-eth";
  if (upperTicker == "FRAX") return "frax";
  if (upperTicker == "GOAT") return "goatseus-maximus";
  if (upperTicker == "JSOL") return "jito-staked-sol";
  if (upperTicker == "NEIRO") return "neiro";
  if (upperTicker == "MANTLE") return "mantle";
  if (upperTicker == "PAXG") return "pax-gold";
  if (upperTicker == "BAND") return "band-protocol";
  if (upperTicker == "QTUM") return "qtum";
  if (upperTicker == "NANO") return "nano";
  if (upperTicker == "ROSE") return "oasis-network";
  if (upperTicker == "WAVES") return "waves";

  // Top 101-200 Cryptocurrencies
  if (upperTicker == "DASH") return "dash";
  if (upperTicker == "ZEC") return "zcash";
  if (upperTicker == "NEO") return "neo";
  if (upperTicker == "EOS") return "eos";
  if (upperTicker == "MINA") return "mina";
  if (upperTicker == "ONE") return "harmony";
  if (upperTicker == "HIVE") return "hive";
  if (upperTicker == "STEEM") return "steem";
  if (upperTicker == "DCR") return "decred";
  if (upperTicker == "ZIL") return "zilliqa";
  if (upperTicker == "ONT") return "ontology";
  if (upperTicker == "ICX") return "icon";
  if (upperTicker == "LSK") return "lisk";
  if (upperTicker == "RVN") return "ravencoin";
  if (upperTicker == "DGB") return "digibyte";
  if (upperTicker == "BAT") return "basic-attention-token";
  if (upperTicker == "ENJ") return "enjin-coin";
  if (upperTicker == "HOT") return "holotoken";
  if (upperTicker == "ANKR") return "ankr";
  if (upperTicker == "LRC") return "loopring";
  if (upperTicker == "STORJ") return "storj";
  if (upperTicker == "DENT") return "dent";
  if (upperTicker == "FUN") return "funfair";
  if (upperTicker == "CELR") return "celer-network";
  if (upperTicker == "COTI") return "coti";
  if (upperTicker == "OCEAN") return "ocean-protocol";
  if (upperTicker == "REEF") return "reef";
  if (upperTicker == "ALICE") return "my-neighbor-alice";
  if (upperTicker == "TLM") return "alien-worlds";
  if (upperTicker == "SLP") return "smooth-love-potion";
  if (upperTicker == "AXS") return "axie-infinity";
  if (upperTicker == "YFI") return "yearn-finance";
  if (upperTicker == "1INCH") return "1inch";
  if (upperTicker == "MKR") return "maker";
  if (upperTicker == "SNX") return "synthetix";
  if (upperTicker == "GMX") return "gmx";
  if (upperTicker == "PERP") return "perpetual-protocol";
  if (upperTicker == "LOOKSRARE") return "looksrare";
  if (upperTicker == "BLUR") return "blur";
  if (upperTicker == "X2Y2") return "x2y2";
  if (upperTicker == "AUDIO") return "audius";
  if (upperTicker == "LIVEPEER") return "livepeer";
  if (upperTicker == "MASK") return "mask-network";
  if (upperTicker == "RALLY") return "rally";
  if (upperTicker == "BADGER") return "badger-dao";
  if (upperTicker == "FARM") return "harvest-finance";
  if (upperTicker == "PICKLE") return "pickle-finance";
  if (upperTicker == "ALPHA") return "alpha-finance";
  if (upperTicker == "CREAM") return "cream-2";
  if (upperTicker == "COVER") return "cover-protocol";
  if (upperTicker == "HEGIC") return "hegic";
  if (upperTicker == "RARI") return "rarible";
  if (upperTicker == "SUPER") return "superrare";
  if (upperTicker == "WHALE") return "whale";
  if (upperTicker == "MEME") return "memecoin";
  if (upperTicker == "DEGEN") return "degen";
  if (upperTicker == "BOME") return "book-of-meme";
  if (upperTicker == "MOG") return "mog-coin";
  if (upperTicker == "POPCAT") return "popcat";

  // Solana Ecosystem
  if (upperTicker == "JITO") return "jito";
  if (upperTicker == "JUP") return "jupiter";
  if (upperTicker == "PYTH") return "pyth";
  if (upperTicker == "DRIFT") return "drift";
  if (upperTicker == "ORCA") return "orca";
  if (upperTicker == "RAY") return "raydium";
  if (upperTicker == "MANGO") return "mango";
  if (upperTicker == "STEP") return "step";
  if (upperTicker == "SERUM") return "serum";
  if (upperTicker == "BONFIDA") return "bonfida";

  
  String lowerTicker = ticker;
  lowerTicker.toLowerCase();
  return lowerTicker;
}




String getCompanyName(const String& symbol) {
  if (finnhub_api_key_config.length() == 0) return symbol;

  String url = "/api/v1/stock/profile2?symbol=" + symbol + "&token=" + finnhub_api_key_config;
  if (!client.connect("finnhub.io", 443)) {
    Serial.println("Failed to connect to Finnhub for company name");
    return symbol;
  }

  client.print(String("GET ") + url + " HTTP/1.1\r\n" + "Host: finnhub.io\r\n" + "Connection: close\r\n\r\n");

  unsigned long timeout = millis() + 5000;
  while (!client.available() && millis() < timeout) {
    delay(10);
  }

  if (!client.available()) {
    client.stop();
    return symbol;
  }

  
  while (client.available()) {
    String line = client.readStringUntil('\n');
    if (line == "\r") break;
  }

  String response = "";
  while (client.available()) {
    response += client.readString();
  }
  client.stop();

  DynamicJsonDocument doc(2048);
  DeserializationError error = deserializeJson(doc, response);
  if (error) {
    Serial.println("JSON Parse Error for company name");
    return symbol;
  }

  if (doc.containsKey("name") && doc["name"].as<String>().length() > 0) {
    String companyName = doc["name"].as<String>();
    Serial.println("Company name found: " + companyName);
    return companyName;
  }

  return symbol;  
}

String getCoinName(const String& coinId) {
  String url = "/api/v3/coins/" + coinId + "?localization=false&tickers=false&market_data=false&community_data=false&developer_data=false";
  if (!client.connect("api.coingecko.com", 443)) {
    Serial.println("Failed to connect to CoinGecko for coin name");
    return coinId;
  }

  client.print(String("GET ") + url + " HTTP/1.0\r\n" +
         "Host: api.coingecko.com\r\n" +
         "User-Agent: ESP32-CYD-Ticker\r\n" +
         "x-cg-demo-api-key: " + coingecko_api_key_config + "\r\n" +
         "Connection: close\r\n\r\n");



  unsigned long timeout = millis() + 5000;
  while (!client.available() && millis() < timeout) {
    delay(10);
  }

  if (!client.available()) {
    client.stop();
    return coinId;
  }

  
  while (client.available()) {
    String line = client.readStringUntil('\n');
    if (line == "\r") break;
  }

  String response = "";
  while (client.available()) {
    response += client.readString();
  }
  client.stop();

  
  String jsonData = extractJSON(response);
  if (jsonData.length() == 0) {
    return coinId;
  }

  DynamicJsonDocument doc(2048);
  DeserializationError error = deserializeJson(doc, jsonData);
  if (error) {
    Serial.println("JSON Parse Error for coin name");
    return coinId;
  }

  if (doc.containsKey("name") && doc["name"].as<String>().length() > 0) {
    String coinName = doc["name"].as<String>();
    Serial.println("Coin name found: " + coinName);
    return coinName;
  }

  return coinId;  
}

unsigned long calculateOptimalInterval() {
  
  int coinGeckoSymbols = 0;
  for (int i = 0; i < numSymbols; i++) {
    if (symbols[i].apiProvider == "coingecko") {
      coinGeckoSymbols++;
    }
  }
  
  if (coinGeckoSymbols == 0) {
    return 30000; 
  }
  
  
  const unsigned long maxCallsPerMonth = 8000; 
  const unsigned long secondsPerMonth = 30 * 24 * 3600;
  
  
  unsigned long targetSecondsPerCall = secondsPerMonth / maxCallsPerMonth;
  
  
  unsigned long safeInterval = (targetSecondsPerCall * coinGeckoSymbols) / numSymbols;
  
  
  safeInterval *= 1000;
  
  safeInterval = max(30000UL, min(600000UL, safeInterval));
  
  Serial.println("=== CORRECTED CALCULATION ===");
  Serial.println("Target seconds per CoinGecko call: " + String(targetSecondsPerCall));
  Serial.println("CoinGecko symbols: " + String(coinGeckoSymbols) + "/" + String(numSymbols));
  Serial.println("Auto-calculated interval: " + String(safeInterval/1000) + " seconds");
  
  return safeInterval;
}



int countCoinGeckoSymbols() {
  int count = 0;
  for (int i = 0; i < numSymbols; i++) {
    if (symbols[i].apiProvider == "coingecko") count++;
  }
  return count;
}

unsigned long estimateMonthlyApiCalls(int coinGeckoSymbols, unsigned long interval) {
  if (coinGeckoSymbols == 0 || numSymbols == 0) return 0;
  
  float coinGeckoRatio = (float)coinGeckoSymbols / numSymbols;
  unsigned long callsPerDay = (24 * 3600 * 1000) / interval * coinGeckoRatio;
  return callsPerDay * 30;
}


void updateSwitchInterval() {
  if (auto_interval_enabled) {
    
    switchInterval = calculateOptimalInterval();
    
    
    Serial.println("=== AUTO INTERVAL ===");
    Serial.println("Symbols: " + String(numSymbols));
    Serial.println("CoinGecko APIs: " + String(countCoinGeckoSymbols()));
    Serial.println("Calculated interval: " + String(switchInterval/1000) + " seconds");
    
  } else {
    
    Serial.println("Using manual interval: " + String(switchInterval/1000) + " seconds");
  }
  
  
  int coinGeckoSymbols = countCoinGeckoSymbols();
  unsigned long callsPerMonth = estimateMonthlyApiCalls(coinGeckoSymbols, switchInterval);
  
  if (callsPerMonth > 9000) {
    Serial.println("⚠️  WARNING: Estimated " + String(callsPerMonth) + " API calls/month!");
    Serial.println("⚠️  Consider increasing interval to stay under 10,000 limit.");
  } else {
    Serial.println("✅ Estimated " + String(callsPerMonth) + " API calls/month - Safe!");
  }
}


void flush_cb(lv_disp_drv_t* disp_drv, const lv_area_t* area, lv_color_t* color_p) {
  uint32_t w = (area->x2 - area->x1 + 1);
  uint32_t h = (area->y2 - area->y1 + 1);

  tft.startWrite();
  tft.setAddrWindow(area->x1, area->y1, w, h);
  tft.pushColors((uint16_t*)&color_p->full, w * h, true);
  tft.endWrite();

  lv_disp_flush_ready(disp_drv);
}

void touchscreen_read(lv_indev_drv_t* indev_drv, lv_indev_data_t* data) {
  if (touchscreen.tirqTouched() && touchscreen.touched()) {
    TS_Point p = touchscreen.getPoint();
    data->point.x = map(p.x, 200, 3800, 0, screenWidth - 1);
    data->point.y = map(p.y, 200, 4000, 0, screenHeight - 1);
    data->point.x = constrain(data->point.x, 0, screenWidth - 1);
    data->point.y = constrain(data->point.y, 0, screenHeight - 1);
    data->state = LV_INDEV_STATE_PR;
  } else {
    data->state = LV_INDEV_STATE_REL;
  }
}

void setupLVGL() {
  lv_init();
  tft.begin();
  tft.setRotation(0);
  tft.fillScreen(TFT_BLACK);

  uint32_t bufSize = screenWidth * 40;

#ifdef CONFIG_SPIRAM_SUPPORT
  if (psramFound()) {
    buf = (lv_color_t*)ps_malloc(bufSize * sizeof(lv_color_t));
  } else {
    buf = (lv_color_t*)malloc(bufSize * sizeof(lv_color_t));
  }
#else
  buf = (lv_color_t*)malloc(bufSize * sizeof(lv_color_t));
#endif

  if (!buf) return;

  lv_disp_draw_buf_init(&draw_buf, buf, NULL, bufSize);

  static lv_disp_drv_t disp_drv;
  lv_disp_drv_init(&disp_drv);
  disp_drv.hor_res = screenWidth;
  disp_drv.ver_res = screenHeight;
  disp_drv.flush_cb = flush_cb;
  disp_drv.draw_buf = &draw_buf;
  disp = lv_disp_drv_register(&disp_drv);
}

void createUI() {
  background_img = lv_img_create(lv_scr_act());
  lv_img_set_src(background_img, &backgroundstonk);
  lv_obj_align(background_img, LV_ALIGN_CENTER, 0, 0);

  symbol_panel = lv_obj_create(lv_scr_act());
  lv_obj_set_size(symbol_panel, LV_SIZE_CONTENT, 32);  
  lv_obj_align(symbol_panel, LV_ALIGN_TOP_LEFT, 15, 25);
  lv_obj_set_style_bg_color(symbol_panel, lv_color_hex(0x000000), 0);
  lv_obj_set_style_bg_opa(symbol_panel, LV_OPA_60, 0);
  lv_obj_set_style_border_width(symbol_panel, 0, 0);
  lv_obj_set_style_radius(symbol_panel, 8, 0);
  lv_obj_set_style_pad_all(symbol_panel, 0, 0);
  lv_obj_clear_flag(symbol_panel, LV_OBJ_FLAG_SCROLLABLE);

  price_panel = lv_obj_create(lv_scr_act());
  lv_obj_set_size(price_panel, 130, 28);
  lv_obj_align(price_panel, LV_ALIGN_TOP_LEFT, 15, 65);
  lv_obj_set_style_bg_color(price_panel, lv_color_hex(0x000000), 0);
  lv_obj_set_style_bg_opa(price_panel, LV_OPA_60, 0);
  lv_obj_set_style_border_width(price_panel, 0, 0);
  lv_obj_set_style_radius(price_panel, 8, 0);
  lv_obj_set_style_pad_all(price_panel, 0, 0);

  percent_panel = lv_obj_create(lv_scr_act());
  lv_obj_set_size(percent_panel, 100, 24);
  lv_obj_align(percent_panel, LV_ALIGN_TOP_LEFT, 15, 100);
  lv_obj_set_style_bg_color(percent_panel, lv_color_hex(0x000000), 0);
  lv_obj_set_style_bg_opa(percent_panel, LV_OPA_60, 0);
  lv_obj_set_style_border_width(percent_panel, 0, 0);
  lv_obj_set_style_radius(percent_panel, 8, 0);
  lv_obj_set_style_pad_all(percent_panel, 0, 0);

  label_symbol = lv_label_create(symbol_panel);
  lv_obj_set_width(label_symbol, screenWidth - 150);
  lv_label_set_long_mode(label_symbol, LV_LABEL_LONG_WRAP);
  lv_obj_set_style_text_font(label_symbol, &lv_font_montserrat_20, 0);   
  lv_obj_set_style_text_color(label_symbol, lv_color_hex(0xFFFFFF), 0);  
  lv_obj_center(label_symbol);
  lv_label_set_text(label_symbol, "LOADING");



  label_price = lv_label_create(price_panel);
  lv_obj_set_style_text_font(label_price, &lv_font_montserrat_18, 0);
  lv_obj_set_style_text_color(label_price, lv_color_hex(0xFFFFFF), 0);
  lv_obj_center(label_price);
  lv_label_set_text(label_price, "$0.00");

  label_percent = lv_label_create(percent_panel);
  lv_obj_set_style_text_font(label_percent, &lv_font_montserrat_16, 0);
  lv_obj_set_style_text_color(label_percent, lv_color_hex(0xFFFFFF), 0);
  lv_obj_center(label_percent);
  lv_label_set_text(label_percent, "(+0.00%)");

  
  lv_obj_t* reset_area = lv_obj_create(lv_scr_act());
  lv_obj_set_pos(reset_area, 0, 0);
  lv_obj_set_size(reset_area, screenWidth, screenHeight);
  lv_obj_set_style_bg_opa(reset_area, LV_OPA_0, LV_PART_MAIN);
  lv_obj_set_style_border_width(reset_area, 0, LV_PART_MAIN);
  lv_obj_add_flag(reset_area, LV_OBJ_FLAG_CLICKABLE);
  lv_obj_add_event_cb(reset_area, longPressResetHandler, LV_EVENT_ALL, NULL);
  lv_obj_move_background(reset_area);  
  
  validation_overlay = lv_obj_create(lv_scr_act());
  lv_obj_set_size(validation_overlay, LV_SIZE_CONTENT, LV_SIZE_CONTENT);  
  lv_obj_align(validation_overlay, LV_ALIGN_CENTER, 0, 0);
  lv_obj_set_style_bg_color(validation_overlay, lv_color_hex(0x000000), 0);
  lv_obj_set_style_bg_opa(validation_overlay, LV_OPA_80, 0);
  lv_obj_set_style_border_width(validation_overlay, 1, 0);
  lv_obj_set_style_border_color(validation_overlay, lv_color_hex(0xFFFFFF), 0);
  lv_obj_set_style_radius(validation_overlay, 8, 0);
  lv_obj_set_style_pad_all(validation_overlay, 8, 0);
  lv_obj_add_flag(validation_overlay, LV_OBJ_FLAG_HIDDEN);

  validation_label = lv_label_create(validation_overlay);
  lv_obj_set_style_text_font(validation_label, &lv_font_montserrat_14, 0);
  lv_obj_set_style_text_color(validation_label, lv_color_hex(0xFFFFFF), 0);
  lv_obj_center(validation_label);
  lv_label_set_text(validation_label, "Validating...");
  lv_label_set_long_mode(validation_label, LV_LABEL_LONG_WRAP);
  lv_obj_set_width(validation_label, LV_SIZE_CONTENT);  
}

void connectWiFi() {
  WiFiManager wm;
  
  
  prefs.begin("config", true);
  String current_finnhub = prefs.getString("finnhub_api", "");
  String current_coingecko = prefs.getString("coingecko_api", "");
  prefs.end();
  
  WiFiManagerParameter custom_finnhub_api("finnhub_api", "Finnhub API Key", current_finnhub.c_str(), 64);
  WiFiManagerParameter custom_coingecko_api("coingecko_api", "CoinGecko API Key", current_coingecko.c_str(), 64);
  WiFiManagerParameter custom_led_toggle("led_toggle", "Status LED Enabled (1=On, 0=Off)", status_led_enabled ? "1" : "0", 2);

 
String dropdownHTML = 
  "<br><b>Switch Interval Configuration:</b><br><br>"
  
  "<label for='interval_mode'>Switch Interval Mode:</label><br>"
  "<select id='interval_mode' name='interval_mode' onchange='toggleManualField()'>"
  "<option value='AUTO'" + String(auto_interval_enabled ? " selected" : "") + ">AUTO (Recommended)</option>"
  "<option value='MANUAL'" + String(!auto_interval_enabled ? " selected" : "") + ">MANUAL</option>"
  "</select><br><br>"
  
  "<div id='manual_div' style='display:" + String(auto_interval_enabled ? "none" : "block") + ";'>"
  "<label for='manual_interval'>Manual Interval (30-600 seconds):</label><br>"
  "<input type='number' id='manual_interval' name='manual_interval' min='30' max='600' value='" + 
  String(auto_interval_enabled ? 60 : switchInterval/1000) + "'><br><br>"
  "</div>"
  
  "<div style='background-color:#f0f0f0; padding:8px; border-radius:4px; font-size:12px;'>"
  "<b>Info:</b><br>"
  "• AUTO mode: Calculates safe intervals automatically<br>"
  "• MANUAL mode: You set the interval (30-600 seconds)<br>"
  "• More cryptos = slower switching recommended<br>"
  "• More stocks = faster switching possible"
  "</div>"
  
  "<script>"
  "function toggleManualField() {"
    "var mode = document.getElementById('interval_mode').value;"
    "var div = document.getElementById('manual_div');"
    "if (mode === 'MANUAL') {"
      "div.style.display = 'block';"
    "} else {"
      "div.style.display = 'none';"
    "}"
  "}"
  "</script>";

WiFiManagerParameter custom_interval_controls(dropdownHTML.c_str());

  
  
  
  WiFiManagerParameter crypto1("crypto1", "Crypto 1", "BTC", 20);
  WiFiManagerParameter crypto2("crypto2", "Crypto 2", "ETH", 20);
  WiFiManagerParameter crypto3("crypto3", "Crypto 3", "DOGE", 20);
  WiFiManagerParameter crypto4("crypto4", "Crypto 4", "TRX", 20);
  WiFiManagerParameter crypto5("crypto5", "Crypto 5", "BNB", 20);

  
  WiFiManagerParameter stock1("stock1", "Stock/ETF 1", "AAPL", 10);
  WiFiManagerParameter stock2("stock2", "Stock/ETF 2", "MSFT", 10);
  WiFiManagerParameter stock3("stock3", "Stock/ETF 3", "NVDA", 10);
  WiFiManagerParameter stock4("stock4", "Stock/ETF 4", "AMZN", 10);
  WiFiManagerParameter stock5("stock5", "Stock/ETF 5", "TSLA", 10);
  WiFiManagerParameter stock6("stock6", "Stock/ETF 6", "GME", 10);
  WiFiManagerParameter stock7("stock7", "Stock/ETF 7", "META", 10);
  WiFiManagerParameter stock8("stock8", "Stock/ETF 8", "GOOGL", 10);
  WiFiManagerParameter stock9("stock9", "Stock/ETF 9", "JPM", 10);
  WiFiManagerParameter stock10("stock10", "Stock/ETF 10", "BNTX", 10);

  
  wm.addParameter(&custom_coingecko_api);
  wm.addParameter(&custom_finnhub_api);
  wm.addParameter(&custom_led_toggle);
  wm.addParameter(&crypto1);
  wm.addParameter(&crypto2);
  wm.addParameter(&crypto3);
  wm.addParameter(&crypto4);
  wm.addParameter(&crypto5);
  wm.addParameter(&stock1);
  wm.addParameter(&stock2);
  wm.addParameter(&stock3);
  wm.addParameter(&stock4);
  wm.addParameter(&stock5);
  wm.addParameter(&stock6);
  wm.addParameter(&stock7);
  wm.addParameter(&stock8);
  wm.addParameter(&stock9);
  wm.addParameter(&stock10);
  wm.addParameter(&custom_interval_controls);
  
  wm.setAPCallback([](WiFiManager* myWiFiManager) {
    displaySetupInstructions(myWiFiManager->getConfigPortalSSID().c_str());
  });
  wm.setConfigPortalTimeout(300);

  
  wm.setSaveConfigCallback([&wm, &custom_finnhub_api, &custom_coingecko_api, &custom_led_toggle, &custom_interval_controls, &crypto1, &crypto2, &crypto3, &crypto4, &crypto5, &stock1, &stock2, &stock3, &stock4, &stock5, &stock6, &stock7, &stock8, &stock9, &stock10]() {
    prefs.begin("config", false);
    
    
    String finnhub_api = custom_finnhub_api.getValue();
    String coingecko_api = custom_coingecko_api.getValue();
    String led_toggle = custom_led_toggle.getValue();
    
    
    Serial.println("Saving API Keys:");
    Serial.println("Finnhub: " + finnhub_api);
    Serial.println("CoinGecko: " + coingecko_api);
    
    
    prefs.putString("finnhub_api", finnhub_api);
    prefs.putString("coingecko_api", coingecko_api);
    
    
    finnhub_api_key_config = finnhub_api;
    coingecko_api_key_config = coingecko_api;
    
    bool new_led_enabled = (led_toggle == "1");
    prefs.putBool("led_enabled", new_led_enabled);
    status_led_enabled = new_led_enabled;
        
        String mode_setting = wm.server->arg("interval_mode");
        String manual_setting = wm.server->arg("manual_interval");

        auto_interval_enabled = (mode_setting == "AUTO");
        prefs.putBool("auto_interval", auto_interval_enabled);

if (!auto_interval_enabled) {
  int manualSeconds = manual_setting.toInt();
  manualSeconds = constrain(manualSeconds, 30, 600);
  switchInterval = manualSeconds * 1000;
  prefs.putULong("switch_interval", switchInterval);
}

Serial.println("Switch settings saved:");
Serial.println("Mode: " + mode_setting);
if (!auto_interval_enabled) {
  Serial.println("Manual interval: " + String(switchInterval/1000) + "s");
}

    prefs.putBool("auto_interval", auto_interval_enabled);

    if (!auto_interval_enabled) {
      int manualSeconds = manual_setting.toInt();
      manualSeconds = constrain(manualSeconds, 30, 600); 
      switchInterval = manualSeconds * 1000;
      prefs.putULong("switch_interval", switchInterval);
    }

    Serial.println("Switch settings saved:");
    Serial.println("Mode: " + String(auto_interval_enabled ? "AUTO" : "MANUAL"));
    if (!auto_interval_enabled) {
      Serial.println("Manual interval: " + String(switchInterval/1000) + "s");
    }

    
    prefs.putString("crypto1", crypto1.getValue());
    prefs.putString("crypto2", crypto2.getValue());
    prefs.putString("crypto3", crypto3.getValue());
    prefs.putString("crypto4", crypto4.getValue());
    prefs.putString("crypto5", crypto5.getValue());
    prefs.putString("stock1", stock1.getValue());
    prefs.putString("stock2", stock2.getValue());
    prefs.putString("stock3", stock3.getValue());
    prefs.putString("stock4", stock4.getValue());
    prefs.putString("stock5", stock5.getValue());
    prefs.putString("stock6", stock6.getValue());
    prefs.putString("stock7", stock7.getValue());
    prefs.putString("stock8", stock8.getValue());
    prefs.putString("stock9", stock9.getValue());
    prefs.putString("stock10", stock10.getValue());
    prefs.end();
    
    
    lv_obj_clean(lv_scr_act());
    lv_obj_t* black_bg = lv_obj_create(lv_scr_act());
    lv_obj_set_size(black_bg, screenWidth, screenHeight);
    lv_obj_set_style_bg_color(black_bg, lv_color_hex(0x000000), 0);
    lv_obj_set_style_bg_opa(black_bg, LV_OPA_100, 0);
    lv_obj_align(black_bg, LV_ALIGN_CENTER, 0, 0);
    lv_obj_t* connecting_label = lv_label_create(lv_scr_act());
    lv_obj_set_style_text_font(connecting_label, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(connecting_label, lv_color_hex(0xFFFFFF), 0);
    lv_obj_align(connecting_label, LV_ALIGN_CENTER, 0, 0);
    lv_label_set_text(connecting_label, "Connecting...");
    lv_obj_invalidate(lv_scr_act());
    lv_refr_now(disp);
  });  

  if (!wm.autoConnect("StockTickerAP", "password")) {
    Serial.println("Failed to connect and hit timeout");
    ESP.restart();
  }

  
  updateStatusLED(WiFi.status() == WL_CONNECTED);
  client.setInsecure();
  createUI();
}


void updateSymbolsFromPreferences() {
  prefs.begin("config", true);

  symbols.clear();
  numSymbols = 0;

  String cryptoSymbols[5] = {
    prefs.getString("crypto1", ""),
    prefs.getString("crypto2", ""),
    prefs.getString("crypto3", ""),
    prefs.getString("crypto4", ""),
    prefs.getString("crypto5", "")
  };

  String stockSymbols[10] = {
    prefs.getString("stock1", ""),
    prefs.getString("stock2", ""),
    prefs.getString("stock3", ""),
    prefs.getString("stock4", ""),
    prefs.getString("stock5", ""),
    prefs.getString("stock6", ""),
    prefs.getString("stock7", ""),
    prefs.getString("stock8", ""),
    prefs.getString("stock9", ""),
    prefs.getString("stock10", "")
  };

  prefs.end();

  
  for (int i = 0; i < 5; i++) {
    if (cryptoSymbols[i].length() > 0) {
      showValidationMessage("Checking " + cryptoSymbols[i]);

      if (validateSymbol(cryptoSymbols[i], true)) {
        SymbolInfo newSymbol;
        newSymbol.originalInput = cryptoSymbols[i];
        newSymbol.coinGeckoId = tickerToCoinGeckoId(cryptoSymbols[i]);
        newSymbol.symbol = newSymbol.coinGeckoId;  
        
        newSymbol.displayName = getCoinName(newSymbol.coinGeckoId);
        newSymbol.apiProvider = "coingecko";
        newSymbol.isCrypto = true;

        symbols.push_back(newSymbol);
        numSymbols++;

        Serial.println("Added crypto: " + cryptoSymbols[i] + " -> " + newSymbol.coinGeckoId + " (" + newSymbol.displayName + ")");
      } else {
        Serial.println("Invalid crypto symbol: " + cryptoSymbols[i]);
      }
    }
  }

  for (int i = 0; i < 10; i++) {
    if (stockSymbols[i].length() > 0) {
      
      showValidationMessage("Checking " + stockSymbols[i]);
      lv_obj_invalidate(lv_scr_act());
      lv_refr_now(disp);

      if (validateSymbol(stockSymbols[i], false)) {
        SymbolInfo newSymbol;
        
        newSymbol.displayName = getCompanyName(stockSymbols[i]);
        newSymbol.symbol = stockSymbols[i];
        newSymbol.apiProvider = "finnhub";
        newSymbol.isCrypto = false;
        symbols.push_back(newSymbol);
        numSymbols++;

        Serial.println("Added stock: " + stockSymbols[i] + " (" + newSymbol.displayName + ")");
      } else {
        Serial.println("Invalid stock symbol: " + stockSymbols[i]);
      }
    }
  }

  
  if (numSymbols == 0) {
    SymbolInfo fallback;
    fallback.displayName = "Bitcoin";
    fallback.symbol = "bitcoin";
    fallback.apiProvider = "coingecko";
    fallback.isCrypto = true;
    symbols.push_back(fallback);
    numSymbols = 1;
    Serial.println("No valid symbols found, using Bitcoin as fallback");
  }

  
  if (marketData) {
    delete[] marketData;
  }
  marketData = new MarketData[numSymbols];

  for (int i = 0; i < numSymbols; i++) {
    marketData[i].dataValid = false;
    marketData[i].isCrypto = symbols[i].isCrypto;
    marketData[i].apiProvider = symbols[i].apiProvider;
    marketData[i].hasBackupData = false;  
  }

  Serial.println("Total valid symbols loaded: " + String(numSymbols));
  
  hideValidationMessage();

  currentSymbolIndex = 0;
  updateSwitchInterval();
  if (numSymbols > 0 && WiFi.status() == WL_CONNECTED) {
  Serial.println("Making initial API call for first symbol...");
  MarketData& initialData = marketData[currentSymbolIndex];
  
  if (fetchMarketData(currentSymbolIndex, initialData)) {
    updateDisplay(initialData);
    Serial.println("✅ Initial display updated with: " + symbols[currentSymbolIndex].displayName);
  } else {
    
    lv_label_set_text(label_symbol, symbols[currentSymbolIndex].displayName.c_str());
    lv_label_set_text(label_price, "Loading...");
    lv_label_set_text(label_percent, "Please wait");
    lv_obj_invalidate(lv_scr_act());
    lv_refr_now(disp);
    
  }
}
}

String extractJSON(const String& response) {
  Serial.println("Extracting JSON from response...");

  
  int jsonStart = response.indexOf('{');
  if (jsonStart >= 0) {
    int jsonEnd = response.lastIndexOf('}');
    if (jsonEnd > jsonStart) {
      String json = response.substring(jsonStart, jsonEnd + 1);
      Serial.println("Found direct JSON: " + json);
      return json;
    }
  }

  String result = "";
  int pos = 0;

  while (pos < response.length()) {
    
    int sizeEnd = response.indexOf("\r\n", pos);
    if (sizeEnd == -1) break;

    String sizeStr = response.substring(pos, sizeEnd);
    sizeStr.trim();

    
    int chunkSize = strtol(sizeStr.c_str(), NULL, 16);

    if (chunkSize == 0) break;

    
    pos = sizeEnd + 2;  
    if (pos + chunkSize <= response.length()) {
      String chunk = response.substring(pos, pos + chunkSize);
      result += chunk;
    }

    pos += chunkSize + 2; 
  }

  
  jsonStart = result.indexOf('{');
  if (jsonStart >= 0) {
    int jsonEnd = result.lastIndexOf('}');
    if (jsonEnd > jsonStart) {
      String json = result.substring(jsonStart, jsonEnd + 1);
      Serial.println("Extracted JSON from chunks: " + json);
      return json;
    }
  }

  Serial.println("No valid JSON found in response");
  return "";
}


bool fetchCoinGeckoData(MarketData& data) {
  String url = "/api/v3/simple/price?ids=" + data.symbol + "&vs_currencies=usd&include_24hr_change=true";
  if (!client.connect("api.coingecko.com", 443)) {
    Serial.println("Verbindung zu CoinGecko fehlgeschlagen");
    return false;
  }

  
  client.print(String("GET ") + url + " HTTP/1.0\r\n" +
           "Host: api.coingecko.com\r\n" +
           "User-Agent: ESP32-CYD-Ticker\r\n" +
           "x-cg-demo-api-key: " + coingecko_api_key_config + "\r\n" +
           "Connection: close\r\n\r\n");


  unsigned long timeout = millis() + 10000;
  while (!client.available() && millis() < timeout) {
    delay(10);
    lv_task_handler();
  }

  if (!client.available()) {
    Serial.println("Timeout: Keine Antwort von CoinGecko");
    client.stop();
    return false;
  }

  
  bool headerEnded = false;
  while (client.available() && !headerEnded) {
    String line = client.readStringUntil('\n');
    if (line == "\r") {
      headerEnded = true;
    }
  }

  
  String response = "";
  while (client.available()) {
    response += client.readString();
  }

  client.stop();
  Serial.println("CoinGecko Roh-Antwort:");
  Serial.println(response);

  
  String jsonData = extractJSON(response);
  Serial.println("Extrahiertes JSON:");
  Serial.println(jsonData);
  if (jsonData.length() == 0) {
    Serial.println("Kein gültiges JSON in der Antwort gefunden");
    return false;
  }

  
  DynamicJsonDocument doc(2048);
  DeserializationError error = deserializeJson(doc, jsonData);
  if (error) {
    Serial.print("JSON Parse Error: ");
    Serial.println(error.c_str());
    return false;
  }

  
  if (doc.containsKey(data.symbol.c_str())) {
    JsonObject crypto = doc[data.symbol.c_str()];
    if (crypto.containsKey("usd") && crypto.containsKey("usd_24h_change")) {
      data.currentPrice = crypto["usd"];
      data.changePercent = crypto["usd_24h_change"];
      data.change = 0;
      data.dataValid = true;
      Serial.printf("CoinGecko Erfolg für %s: $%.2f (%.2f%%)\n",
                    data.symbol.c_str(), data.currentPrice, data.changePercent);
      return true;
    }
  }

  Serial.println("CoinGecko: Erwartete Daten nicht in der JSON-Antwort");
  return false;
}



bool fetchFinnhubData(const char* symbol, MarketData& data) {
  String url = "https://finnhub.io/api/v1/quote?symbol=" + String(symbol) + "&token=" + finnhub_api_key_config;

  if (!client.connect("finnhub.io", 443)) {
    return false;
  }

  client.print(String("GET ") + url + " HTTP/1.1\r\n" + "Host: finnhub.io\r\n" + "Connection: close\r\n\r\n");

  unsigned long timeout = millis() + 10000;
  while (!client.available() && millis() < timeout) {
    delay(10);
    lv_task_handler();
  }

  if (!client.available()) {
    client.stop();
    return false;
  }

  while (client.available()) {
    String line = client.readStringUntil('\n');
    if (line == "\r") {
      break;
    }
  }

  String response = "";
  while (client.available()) {
    response += client.readString();
  }
  client.stop();

  DynamicJsonDocument doc(1024);
  DeserializationError error = deserializeJson(doc, response);

  if (error) return false;

  if (doc.containsKey("c") && doc.containsKey("dp")) {
    data.currentPrice = doc["c"];
    data.changePercent = doc["dp"];
    data.change = doc["d"];
    data.dataValid = true;
    return true;
  }

  return false;
}

bool fetchMarketData(int index, MarketData& data) {
  if (index >= numSymbols || index < 0) {
    Serial.println("Invalid symbol index: " + String(index));
    return false;
  }

  data.displayName = symbols[index].displayName;
  data.symbol = symbols[index].symbol;
  data.isCrypto = symbols[index].isCrypto;
  data.apiProvider = symbols[index].apiProvider;

  Serial.println("Fetching data for: " + data.symbol + " (" + data.apiProvider + ")");

  if (symbols[index].apiProvider == "coingecko") {
    return fetchCoinGeckoData(data);
  } else if (symbols[index].apiProvider == "finnhub") {
    return fetchFinnhubData(symbols[index].symbol.c_str(), data);
  }

  return false;
}



void updateDisplay(const MarketData& data) {
  lv_label_set_text(label_symbol, data.displayName.c_str());
  lv_obj_set_style_text_color(label_symbol, lv_color_hex(0xFFFFFF), 0);  

  
  lv_obj_set_width(label_symbol, LV_SIZE_CONTENT);  
  lv_obj_update_layout(label_symbol);               
  lv_obj_set_size(symbol_panel, lv_obj_get_width(label_symbol) + 20, lv_obj_get_height(label_symbol) + 10);
  lv_obj_update_layout(symbol_panel);  



  if (data.dataValid) {
    
    char priceStr[20];
    snprintf(priceStr, sizeof(priceStr), "$%.2f", data.currentPrice);
    lv_label_set_text(label_price, priceStr);

    char percentStr[20];
    snprintf(percentStr, sizeof(percentStr), "(%+.2f%%)", data.changePercent);
    lv_label_set_text(label_percent, percentStr);

    lv_color_t change_color;
    if (data.changePercent > 0) {
      change_color = lv_color_hex(0x00FF88);
    } else if (data.changePercent < 0) {
      change_color = lv_color_hex(0xFF4444);
    } else {
      change_color = lv_color_hex(0xFFFFFF);
    }

    lv_obj_set_style_text_color(label_price, change_color, 0);
    lv_obj_set_style_text_color(label_percent, change_color, 0);

  } else if (data.hasBackupData) {
    
    char priceStr[25];
    snprintf(priceStr, sizeof(priceStr), "$%.2f*", data.lastValidPrice);
    lv_label_set_text(label_price, priceStr);

    char percentStr[25];
    snprintf(percentStr, sizeof(percentStr), "(%+.2f%%)*", data.lastValidChangePercent);
    lv_label_set_text(label_percent, percentStr);

    
    lv_color_t stale_color = lv_color_hex(0xFFAA00);
    lv_obj_set_style_text_color(label_price, stale_color, 0);
    lv_obj_set_style_text_color(label_percent, stale_color, 0);

  } else {
    lv_label_set_text(label_price, "No Data");
    lv_label_set_text(label_percent, "(--)");

    lv_color_t error_color = lv_color_hex(0xFF4444);
    lv_obj_set_style_text_color(label_price, error_color, 0);
    lv_obj_set_style_text_color(label_percent, error_color, 0);
  }

  lv_obj_invalidate(lv_scr_act());
  lv_refr_now(disp);
}


void displaySetupInstructions(const char* ssid) {
  
  lv_obj_clean(lv_scr_act());  

  lv_obj_t* black_bg = lv_obj_create(lv_scr_act());
  lv_obj_set_size(black_bg, screenWidth, screenHeight);
  lv_obj_set_style_bg_color(black_bg, lv_color_hex(0x000000), 0);
  lv_obj_set_style_bg_opa(black_bg, LV_OPA_100, 0);
  lv_obj_align(black_bg, LV_ALIGN_CENTER, 0, 0);

  
  lv_obj_t* title_label = lv_label_create(lv_scr_act());
  lv_obj_set_style_text_font(title_label, &lv_font_montserrat_20, 0);
  lv_obj_set_style_text_color(title_label, lv_color_hex(0xFFFFFF), 0);
  lv_obj_align(title_label, LV_ALIGN_TOP_MID, 0, 10);
  lv_label_set_text(title_label, "WiFi Setup Required");

  
  lv_obj_t* instr_label = lv_label_create(lv_scr_act());
  lv_obj_set_style_text_font(instr_label, &lv_font_montserrat_14, 0);
  lv_obj_set_style_text_color(instr_label, lv_color_hex(0xFFFFFF), 0);
  lv_obj_align(instr_label, LV_ALIGN_TOP_MID, 0, 40);
  lv_obj_set_width(instr_label, screenWidth - 20);  
  lv_label_set_long_mode(instr_label, LV_LABEL_LONG_WRAP);
  char instr_text[256];
  snprintf(instr_text, sizeof(instr_text),
           "1. Connect to WiFi: %s\n2. Open browser, go to 192.168.4.1\n3. Enter WiFi details & API key\n4. Save to connect",
           ssid);
  lv_label_set_text(instr_label, instr_text);

  
  lv_obj_invalidate(lv_scr_act());
  lv_refr_now(disp);
}

void updateStatusLED(bool wifiConnected) {
  if (!status_led_enabled) {
    
    digitalWrite(STATUS_LED_RED_PIN, HIGH);    
    digitalWrite(STATUS_LED_GREEN_PIN, HIGH);  
    digitalWrite(STATUS_LED_BLUE_PIN, HIGH);   
  } else {
    if (wifiConnected) {
      
      digitalWrite(STATUS_LED_RED_PIN, HIGH);   
      digitalWrite(STATUS_LED_GREEN_PIN, LOW);  
      digitalWrite(STATUS_LED_BLUE_PIN, HIGH);  
    } else {
      
      digitalWrite(STATUS_LED_RED_PIN, LOW);     
      digitalWrite(STATUS_LED_GREEN_PIN, HIGH);  
      digitalWrite(STATUS_LED_BLUE_PIN, HIGH);   
    }
  }
}


void showValidationMessage(const String& message) {
  if (validation_overlay && validation_label) {
    lv_label_set_text(validation_label, message.c_str());

    
    lv_obj_set_width(validation_label, LV_SIZE_CONTENT);
    lv_obj_update_layout(validation_label);  

    
    int textWidth = lv_obj_get_width(validation_label);
    int panelWidth = textWidth + 16;  
    lv_obj_set_width(validation_overlay, panelWidth);

    lv_obj_clear_flag(validation_overlay, LV_OBJ_FLAG_HIDDEN);
    lv_obj_invalidate(lv_scr_act());
    lv_refr_now(disp);
    delay(100);
  }
}

void hideValidationMessage() {
  if (validation_overlay) {
    lv_obj_add_flag(validation_overlay, LV_OBJ_FLAG_HIDDEN);
    lv_obj_invalidate(lv_scr_act());
    lv_refr_now(disp);
  }
}


void longPressResetHandler(lv_event_t* e) {
  uint32_t eventCode = lv_event_get_code(e);
  
  if (eventCode == LV_EVENT_PRESSED && !resetInProgress) {
    touchPressStart = millis();
    touchPressed = true;
    Serial.println("Touch started - hold for 10s to reset");
    
  } else if (eventCode == LV_EVENT_RELEASED) {
    touchPressed = false;
    if (!resetInProgress && numSymbols > 0 && marketData[currentSymbolIndex].dataValid) {
      updateDisplay(marketData[currentSymbolIndex]);
    }
  }
}

void setup() {
  Serial.begin(115200);

  
  pinMode(STATUS_LED_RED_PIN, OUTPUT);
  pinMode(STATUS_LED_GREEN_PIN, OUTPUT);
  pinMode(STATUS_LED_BLUE_PIN, OUTPUT);
  digitalWrite(STATUS_LED_RED_PIN, HIGH);
  digitalWrite(STATUS_LED_GREEN_PIN, HIGH);
  digitalWrite(STATUS_LED_BLUE_PIN, HIGH);

  
  prefs.begin("config", true);
  finnhub_api_key_config = prefs.getString("finnhub_api", "");
  coingecko_api_key_config = prefs.getString("coingecko_api", "");
  status_led_enabled = prefs.getBool("led_enabled", true);
  auto_interval_enabled = prefs.getBool("auto_interval", true);
  switchInterval = prefs.getULong("switch_interval", 25000);
  prefs.end();

  
  Serial.println("Loaded from Preferences:");
  Serial.println("Finnhub API Key: '" + finnhub_api_key_config + "'");
  Serial.println("CoinGecko API Key: '" + coingecko_api_key_config + "'");

  updateStatusLED(WiFi.status() == WL_CONNECTED);
  setupLVGL();

  touchscreenSPI.begin(XPT2046_CLK, XPT2046_MISO, XPT2046_MOSI, XPT2046_CS);
  touchscreen.begin(touchscreenSPI);
  touchscreen.setRotation(0);

  lv_indev_drv_init(&indev_drv);
  indev_drv.type = LV_INDEV_TYPE_POINTER;
  indev_drv.read_cb = touchscreen_read;
  lv_indev_drv_register(&indev_drv);

  createUI();
  connectWiFi();

  
  updateSymbolsFromPreferences();

  lastSwitchTime = millis();
  
}




void loop() {
  lv_task_handler();
  lv_tick_inc(2);
  updateStatusLED(WiFi.status() == WL_CONNECTED);

  
  if (touchPressed && !resetInProgress) {
    unsigned long pressDuration = millis() - touchPressStart;
    
    if (pressDuration >= 10000) {
      resetInProgress = true;
      touchPressed = false;
      
      Serial.println("10s long press detected - resetting WiFi");
      lv_label_set_text(label_symbol, "RESETTING...");
      lv_label_set_text(label_price, "WIFI CONFIG");
      lv_label_set_text(label_percent, "PLEASE WAIT");
      lv_obj_invalidate(lv_scr_act());
      lv_refr_now(disp);
      
      delay(2000);
      WiFi.disconnect(true);
      WiFiManager wm;
      wm.resetSettings();
      prefs.begin("config", false);
      prefs.clear();
      prefs.end();
      ESP.restart();
    }
  }

  unsigned long currentTime = millis();

  
  if (currentTime - lastSwitchTime >= switchInterval && numSymbols > 0) {
    currentSymbolIndex = (currentSymbolIndex + 1) % numSymbols;
    
    Serial.println("=== Symbol Switch ===");
    Serial.println("New symbol: " + symbols[currentSymbolIndex].displayName);

    if (WiFi.status() == WL_CONNECTED) {
      
      MarketData& currentData = marketData[currentSymbolIndex];
      
      if (fetchMarketData(currentSymbolIndex, currentData)) {
        
        currentData.lastValidPrice = currentData.currentPrice;
        currentData.lastValidChange = currentData.change;
        currentData.lastValidChangePercent = currentData.changePercent;
        currentData.hasBackupData = true;
        
        updateDisplay(currentData);
        Serial.println("✅ API call successful for " + symbols[currentSymbolIndex].displayName);
      } else if (currentData.hasBackupData) {
        
        updateDisplay(currentData);  
        Serial.println("⚠️ API failed, showing backup data for " + symbols[currentSymbolIndex].displayName);
      } else {
        
        Serial.println("❌ API call failed, no backup data for " + symbols[currentSymbolIndex].displayName);
        lv_label_set_text(label_symbol, symbols[currentSymbolIndex].displayName.c_str());
        lv_label_set_text(label_price, "API Error");
        lv_label_set_text(label_percent, "(retrying next switch)");
        lv_obj_invalidate(lv_scr_act());
        lv_refr_now(disp);
      }
    } else {
      
      lv_label_set_text(label_symbol, symbols[currentSymbolIndex].displayName.c_str());
      lv_label_set_text(label_price, "No WiFi");
      lv_label_set_text(label_percent, "(connecting...)");
      lv_obj_invalidate(lv_scr_act());
      lv_refr_now(disp);
      
      
      if (currentTime % 30000 < 100) {
        connectWiFi();
      }
    }

    lastSwitchTime = currentTime;  
  }

  delay(2);
}


