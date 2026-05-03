// ==================================================================
//               EFIS MINIMALISTE - ESP32-S3 & LOVYANGFX        
// ==================================================================

#define LGFX_USE_V1
#include <LovyanGFX.hpp>
#include <math.h> 

// ------------------------------------------------------------------
// 1. CONFIGURATION DU MATÉRIEL (ESP32-S3 + Écran ILI9341)
// ------------------------------------------------------------------
class LGFX : public lgfx::LGFX_Device {
  lgfx::Panel_ILI9341 _panel_instance; 
  lgfx::Bus_SPI       _bus_instance;   

public:
  LGFX(void) {
    { 
      auto cfg = _bus_instance.config();
      cfg.spi_host = SPI2_HOST;  
      cfg.spi_mode = 0;          
      cfg.freq_write = 40000000; 
      cfg.pin_sclk = 12;         
      cfg.pin_mosi = 11;         
      cfg.pin_miso = 13;         
      cfg.pin_dc   = 46;         
      _bus_instance.config(cfg);
      _panel_instance.setBus(&_bus_instance);
    }
    { 
      auto cfg = _panel_instance.config();
      cfg.pin_cs           = 10;   
      cfg.pin_rst          = -1;   
      cfg.panel_width      = 240;  
      cfg.panel_height     = 320;  
      cfg.offset_rotation  = 3;    // Paysage inversé (Ciel en haut)
      cfg.invert           = true; // Correction colorimétrique
      _panel_instance.config(cfg);
    }
    setPanel(&_panel_instance);
  }
};

LGFX tft;               // Écran physique
LGFX_Sprite spr(&tft);  // Écran virtuel en RAM (Sprite)

// --- BROCHES ET COULEURS ---
#define RX_PIN RX 
#define TX_PIN TX 
#define SKY_BLUE    0x03FF 
#define BROWN_EARTH 0x8200 
#define ORANGE_ACFT 0xFD20 

// --- VARIABLES GLOBALES ---
byte buffer[74];               
unsigned long dernierAppel = 0; 
float roll = 0, pitch = 0, nbSat = 0; // Seules les 3 variables utiles sont conservées


// ------------------------------------------------------------------
// 2. INITIALISATION
// ------------------------------------------------------------------
void setup() {
  Serial.begin(115200);                             
  Serial2.begin(115200, SERIAL_8N1, RX_PIN, TX_PIN); 

  // Allumage direct de l'écran
  pinMode(45, OUTPUT);
  digitalWrite(45, HIGH); 
  tft.init();

  // Création du Sprite graphique anti-clignotement
  spr.setColorDepth(8); 
  if (spr.createSprite(320, 240) == nullptr) {
    tft.fillScreen(0x0000);
    tft.setCursor(10,10); tft.setTextColor(0xFFFF);
    tft.println("Erreur RAM Sprite");
    while(1); 
  }
  
  spr.fillSprite(SKY_BLUE); 
  spr.pushSprite(0, 0);     
}


// ------------------------------------------------------------------
// 3. MOTEUR GRAPHIQUE : HORIZON & ASSIETTE
// ------------------------------------------------------------------
void dessinerHorizon(float r, float p) {
  int cx = 160, cy = 120; 
  float pitchOff = p * 150.0; 
  float angle = -r;
  float cosR = cos(angle), sinR = sin(angle);
  
  spr.fillSprite(SKY_BLUE); // Efface l'image précédente
  
  // Polygone de la Terre
  float L = 400; 
  int hx1 = cx + (-L*cosR - pitchOff*sinR); int hy1 = cy + (-L*sinR + pitchOff*cosR);
  int hx2 = cx + (L*cosR - pitchOff*sinR);  int hy2 = cy + (L*sinR + pitchOff*cosR);
  int tx1 = cx + (-L*cosR - (pitchOff+L)*sinR); int ty1 = cy + (-L*sinR + (pitchOff+L)*cosR);
  int tx2 = cx + (L*cosR - (pitchOff+L)*sinR);  int ty2 = cy + (L*sinR + (pitchOff+L)*cosR);
  
  spr.fillTriangle(hx1, hy1, hx2, hy2, tx1, ty1, BROWN_EARTH);
  spr.fillTriangle(hx2, hy2, tx1, ty1, tx2, ty2, BROWN_EARTH);
  spr.drawLine(hx1, hy1, hx2, hy2, 0xFFFF);
  
  // Lignes d'assiette de 5° à 30°
  for (int i = 5; i <= 30; i += 5) {
    float rad = i * PI / 180.0; 
    int largeur = (i % 10 == 0) ? 30 : 15; 
    
    dessinerLignePitch(cx, cy, cosR, sinR, pitchOff, rad, largeur);
    dessinerLignePitch(cx, cy, cosR, sinR, pitchOff, -rad, largeur);
  }
}

void dessinerLignePitch(int cx, int cy, float cosR, float sinR, float offsetActuel, float radCible, int largeur) {
  float dY = offsetActuel - (radCible * 150.0); 
  
  // Masquage pour ne pas toucher le Sky Pointer
  if (dY < -80 || dY > 150) return; 
  
  int xA = cx + (-largeur * cosR - dY * sinR);
  int yA = cy + (-largeur * sinR + dY * cosR);
  int xB = cx + (largeur * cosR - dY * sinR);
  int yB = cy + (largeur * sinR + dY * cosR);
  
  spr.drawLine(xA, yA, xB, yB, 0xFFFF); 
}


// ------------------------------------------------------------------
// 4. MOTEUR GRAPHIQUE : SKY POINTER
// ------------------------------------------------------------------
void dessinerEchelleRoulis(float r) {
  int cx = 160, cy = 120;
  int R = 105; 

  spr.drawArc(cx, cy, R, R + 1, 210, 330, 0xFFFF);

  int angles[] = {-60, -45, -30, -20, -10, 0, 10, 20, 30, 45, 60};
  
  for (int i = 0; i < 11; i++) {
    int angleAbsolu = abs(angles[i]); 
    if (angleAbsolu == 0) continue; 
    
    float rad = (angles[i] - 90) * PI / 180.0; 
    
    if (angleAbsolu == 45) {
      int dotR = R + 4; 
      spr.fillCircle(cx + dotR * cos(rad), cy + dotR * sin(rad), 2, 0xFFFF); 
    } 
    else {
      int outerR = (angleAbsolu == 30 || angleAbsolu == 60) ? R + 10 : R + 5; 
      spr.drawLine(cx + R * cos(rad), cy + R * sin(rad), cx + outerR * cos(rad), cy + outerR * sin(rad), 0xFFFF);
    }
  }

  // Triangle fixe (Zénith 0°)
  float fixRad = -PI/2; 
  spr.fillTriangle(
    cx + R * cos(fixRad), cy + R * sin(fixRad), 
    cx + (R + 8) * cos(fixRad - 0.05), cy + (R + 8) * sin(fixRad - 0.05), 
    cx + (R + 8) * cos(fixRad + 0.05), cy + (R + 8) * sin(fixRad + 0.05), 
    0xFFFF
  ); 

  // Pointeur mobile (solidaire de l'horizon)
  float pointerRad = -PI/2 - r; 
  int pointeurR = R - 1; 
  spr.fillTriangle(
    cx + pointeurR * cos(pointerRad), cy + pointeurR * sin(pointerRad), 
    cx + (pointeurR - 10) * cos(pointerRad - 0.06), cy + (pointeurR - 10) * sin(pointerRad - 0.06), 
    cx + (pointeurR - 10) * cos(pointerRad + 0.06), cy + (pointeurR - 10) * sin(pointerRad + 0.06), 
    ORANGE_ACFT
  ); 
}


// ------------------------------------------------------------------
// 5. MOTEUR GRAPHIQUE : MAQUETTE AVION & HUD
// ------------------------------------------------------------------
void dessinerMaquetteAvion() {
  int cx = 160, cy = 120;
  
  // Silhouette noire (Ombre/Contour)
  spr.fillRect(cx - 61, cy - 2, 42, 5, 0x0000); 
  spr.fillRect(cx - 21, cy - 2, 5, 12, 0x0000); 
  spr.fillRect(cx + 19, cy - 2, 42, 5, 0x0000); 
  spr.fillRect(cx + 16, cy - 2, 5, 12, 0x0000); 
  spr.fillCircle(cx, cy, 4, 0x0000);            

  // Cœur orange
  spr.fillRect(cx - 60, cy - 1, 40, 3, ORANGE_ACFT); 
  spr.fillRect(cx - 20, cy - 1, 3, 10, ORANGE_ACFT); 
  spr.fillRect(cx + 20, cy - 1, 40, 3, ORANGE_ACFT); 
  spr.fillRect(cx + 17, cy - 1, 3, 10, ORANGE_ACFT); 
  spr.fillCircle(cx, cy, 3, ORANGE_ACFT);            
}

void dessinerHUD() {
  int s = (int)nbSat;
  for(int i=0; i<5; i++) {
    uint16_t col = (s > i*3) ? 0x07E0 : 0x7BEF;
    spr.fillRect(280 + (i*6), 20 - (i*3), 4, (i*3)+5, col);
  }
}


// ------------------------------------------------------------------
// 6. BOUCLE PRINCIPALE
// ------------------------------------------------------------------
void loop() {
  // 1. Demande de trame au capteur (~25 FPS)
  if (millis() - dernierAppel > 40) {
    Serial2.print('F');
    dernierAppel = millis();
  }

  // 2. Réception et affichage immédiat
  if (Serial2.available() >= 74) {
    if (Serial2.read() == 'F') { 
      
      Serial2.readBytes(&buffer[1], 73); 
      memcpy(&roll,  &buffer[2], 4); 
      memcpy(&pitch, &buffer[6], 4); 
      memcpy(&nbSat, &buffer[70], 4); 
      
      while(Serial2.available()) Serial2.read();

      // 3. Rendu 100% Fluide : 
      dessinerHorizon(roll, pitch); 
      dessinerEchelleRoulis(roll);  
      dessinerMaquetteAvion();      
      dessinerHUD();                
      
      // Envoi du Sprite sur l'écran d'un seul coup
      spr.pushSprite(0, 0); 
    }
  }
}
