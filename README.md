#include <WiFi.h>
#include <WebServer.h>
#include <BluetoothSerial.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_Fingerprint.h>
#include <Preferences.h>

/* Wi-Fi soft-AP credentials */
const char* WIFI_SSID     = "ESP32_VOTE";
const char* WIFI_PASSWORD = "vote1234";

/* built-in synchronous web server */
WebServer server(80);


// ── HARDWARE PINS ─────────────────────────────────────────────
BluetoothSerial    SerialBT;
LiquidCrystal_I2C  lcd(0x27, 20, 4);
Preferences        preferences;               // NVS flash

const int BTN_OK   = 13;
const int BTN_CNC  = 12;
const int BTN_UP   = 14;
const int BTN_DWN  = 27;
const int BTN_LFT  = 26;      // ← returned for completeness
const int BTN_RHT  = 25;      // ← returned for completeness
const int LED_PIN  = 2;
#define FINGER_RX_PIN 16
#define FINGER_TX_PIN 17

// >>>  Put this near the top of your file  <<<
#define SENSOR_CAP  127          // real capacity of your sensor (1-127, 1-200…)
static const int MIN_FID = 1;    // allowed range now equals sensor slots
static const int MAX_FID = SENSOR_CAP;
static const int MAX_VOTERS = MAX_FID + 1;
Adafruit_Fingerprint finger(&Serial2);

// ── CONFIG ────────────────────────────────────────────────────
static const String adminPass = "admin123";      // Bluetooth admin login
static const String candPass  = "candpass123";// Password to edit candidates
static const String fpPass    = "fppass123"; // password to give FF
static const int    maxTries  = 3;               // admin login attempts


// ── GLOBAL VARIABLES ─────────────────────────────────────────
String   candidates[8];
uint8_t  candCount     = 0;
int      votesTally[8] = {0};
bool     votedIDs[MAX_VOTERS] = {0};
bool     votingClosed  = false;

int      menuIndex     = 0;      // 0…3 for main menu
 

// ── HELPER FUNCTIONS ─────────────────────────────────────────
// read a button (LOW when pressed)
bool pressed(int pin) {
  return digitalRead(pin) == LOW;
}
// wait until CANCEL is pressed & released
void waitForCancel() {
  while (!pressed(BTN_CNC)) delay(20);
  while ( pressed(BTN_CNC)) delay(20);
}
// ─────────────────────────────────────────────────────────────
// builds a tiny HTML page that shows the four main-menu items
// the one currently selected menuIndex is highlighted
// ─────────────────────────────────────────────────────────────
String buildMenuPage()
{
  /* strings WITHOUT the leading 1. 2. 3. 4. */
  static const char* items[4] = {
    "Save Candidates",
    "Save Fingerprint",
    "Go for vote",
    "Show Winner"
  };

  String page =
    "<!DOCTYPE html><html><head>"
    "<meta http-equiv='refresh' content='1'>"
    "<meta charset='UTF-8'>"
    "<style>body{font-family:sans-serif}.sel{font-weight:bold;color:#090}</style>"
    "<title>Voting Machine</title></head><body>"
    "<h2>Voting Machine – Main Menu</h2><ol>";

  for (int i = 0; i < 4; ++i) {
    page += "<li class='";
    page += (i == menuIndex) ? "sel'>" : "'>";
    page += items[i];            //   <-- now prints e.g. "Save Candidates"
    page += "</li>";
  }

  page += "</ol></body></html>";
  return page;
}
// redraw the 4-item main menu
void drawMainMenu() {
  const char* items[4] = {
    "1.Save Candidates",
    "2.Save Fingerprint",
    "3.Go for vote",
    "4.Show Winner"
  };
  lcd.clear();
  for (int i = 0; i < 4; ++i) {
    lcd.setCursor(0, i);
    lcd.print(i == menuIndex ? '>' : ' ');
    lcd.print(items[i]);
  }
}

// fingerprint authentication: returns fingerID or ‑1
int authenticate() {
  lcd.clear(); lcd.setCursor(0, 0);
  lcd.print("Place finger...");
  unsigned long start = millis();

  while (millis() - start < 5000) {
    if (finger.getImage() == FINGERPRINT_OK) {
      finger.image2Tz();
      if (finger.fingerFastSearch() == FINGERPRINT_OK) {
        int fid = finger.fingerID;
        lcd.clear(); lcd.setCursor(0, 1);
        lcd.print("ID="); lcd.print(fid);
        delay(800);
        return fid;
      }
      break;
    }
    delay(100);
  }
  lcd.clear(); lcd.setCursor(0, 1);
  lcd.print("Auth Failed");
  delay(800);
  return -1;
}

// ── MENU ACTIONS ─────────────────────────────────────────────
// 1) Load candidate names — password protected & stored in flash
// 1) Load candidate names — password protected & stored in flash
void saveCandidates() {

  // ── PASSWORD STEP ────────────────────────────────────────
  SerialBT.println("ENTER CAND PASSWORD:");
  SerialBT.printf("(press HW CANCEL or type CANCEL to abort, %d tries)\n",
                  maxTries);
  lcd.clear(); lcd.setCursor(0, 1);
  lcd.print("Cand PW ?");

  String in;
  uint8_t passTries = 0;

  while (true) {

    // a) hardware CANCEL button aborts immediately
    if (pressed(BTN_CNC)) {
      SerialBT.println("CANCELLED (button)");
      waitForCancel();          // be sure the button is released
      drawMainMenu();
      return;
    }

    // b) check Bluetooth input
    if (SerialBT.available()) {
      in = SerialBT.readStringUntil('\n'); in.trim();

      // user typed “CANCEL”
      if (in.equalsIgnoreCase("CANCEL")) {
        SerialBT.println("CANCELLED");
        drawMainMenu();
        return;
      }

      // correct password
      if (in == candPass) {
        SerialBT.println("OK");
        break;                  // → continue to candidate entry
      }

      // wrong password
      passTries++;
      if (passTries >= maxTries) {
        SerialBT.println("TOO MANY FAILS — ABORT");
        lcd.clear(); lcd.setCursor(0, 1);
        lcd.print("Too many fails");
        delay(1000);
        drawMainMenu();
        return;
      }
      SerialBT.printf("WRONG (%d/%d). Try again or CANCEL\n",
                      passTries, maxTries);
    }
    delay(50);
  }

  // ── ENTER NAMES ──────────────────────────────────────────
  SerialBT.println("ENTER UP TO 8 NAMES. Send DONE to finish.");
  lcd.clear(); lcd.setCursor(0, 1);
  lcd.print("BT: send names");

  candCount = 0;
  while (candCount < 8) {
    SerialBT.printf("NAME#%d:\n", candCount + 1);
    String nm;
    while (nm.length() == 0) {
      if (SerialBT.available()) {
        nm = SerialBT.readStringUntil('\n'); nm.trim();
      }
      delay(50);
    }
    if (nm.equalsIgnoreCase("DONE")) break;
    candidates[candCount++] = nm;
    SerialBT.println("OK");
  }
  if (candCount == 0) {                // safety: at least one name
    candidates[0] = "NoName";
    candCount = 1;
  }

  // reset votes & who-voted list
  for (int i = 0; i < candCount; ++i) votesTally[i] = 0;
  memset(votedIDs, 0, sizeof(votedIDs));

  // ── STORE IN FLASH ───────────────────────────────────────
  preferences.begin("vote", false);
  preferences.putUChar("candCount", candCount);
  for (int i = 0; i < candCount; ++i) {
    preferences.putString( ("cand" + String(i)).c_str(),
                           candidates[i].c_str() );
  }
  preferences.end();

  // feedback
  lcd.clear(); lcd.setCursor(2, 1);
  lcd.print("Cands Saved!");
  delay(800);
  lcd.clear(); lcd.setCursor(0, 1);
  lcd.print("Press CANCEL");
  waitForCancel();
  drawMainMenu();
}
// 2) Enroll a new fingerprint template
// 2)  SAVE / ENROL FINGERPRINT  (completely rewritten)

// ─────────────────────────────────────────────────────────────

// 2)  SAVE / ENROL FINGERPRINT  (now prevents duplicates)
// ────────────────────────────────────────────────────────────────
// 2)  ENROL / SAVE FINGERPRINT(S)
// ────────────────────────────────────────────────────────────────
// • asks password once
// • afterwards lets operator add many fingerprints in a row
//   OK     = add next one
//   CANCEL = leave to main menu
// • prevents duplicates (same finger in several IDs)

void enrollFingerprint()
{
  /* ───────────── PASSWORD (asked only once) ────────────── */
  SerialBT.printf("ENTER FP PASSWORD (%d tries, CANCEL=abort):\n", maxTries);
  lcd.clear(); lcd.setCursor(0, 1); lcd.print("FP PW ?");
  String in;  uint8_t tries = 0;

  while (true) {
    if (pressed(BTN_CNC)) {              // HW cancel
      SerialBT.println("CANCELLED (button)");
      waitForCancel(); drawMainMenu(); return;
    }
    if (SerialBT.available()) {
      in = SerialBT.readStringUntil('\n'); in.trim();
      if (in.equalsIgnoreCase("CANCEL")) {
        SerialBT.println("CANCELLED");  drawMainMenu(); return;
      }
      if (in == fpPass) { SerialBT.println("OK"); break; }   // success
      if (++tries >= maxTries) {
        SerialBT.println("TOO MANY FAILS — ABORT");
        lcd.clear(); lcd.setCursor(0, 1); lcd.print("Too many fails");
        delay(1000); drawMainMenu(); return;
      }
      SerialBT.printf("WRONG (%d/%d)\n", tries, maxTries);
    }
    delay(50);
  }


  /* ───────────── MAIN ENROL-LOOP ───────────── */
  while (true) {

    /* ---- 1) ask for a free ID ---------------- */
    SerialBT.printf("\nENTER NEW FINGER ID (%d-%d)  (type CANCEL to stop):\n",
                    MIN_FID, MAX_FID);
    lcd.clear(); lcd.setCursor(0, 0); lcd.print("BT: send ID");
    int newID = 0;

    while (true) {
      if (pressed(BTN_CNC)) goto exit_enrol;        // leave whole routine

      if (SerialBT.available()) {
        String s = SerialBT.readStringUntil('\n'); s.trim();
        if (s.equalsIgnoreCase("CANCEL")) goto exit_enrol;

        newID = s.toInt();
        if (newID < MIN_FID || newID > MAX_FID) {
          SerialBT.printf("BAD RANGE %d-%d – try again:\n", MIN_FID, MAX_FID);
          newID = 0;  continue;
        }
        if (finger.loadModel(newID) == FINGERPRINT_OK) {
          SerialBT.println("ID ALREADY USED – choose another:");
          lcd.clear(); lcd.setCursor(0,1); lcd.print("ID already used");
          delay(800);  lcd.clear(); lcd.setCursor(0,0); lcd.print("BT: send ID");
          newID = 0;  continue;
        }
        SerialBT.printf("Enrolling ID=%d\n", newID);
        break;                      // good ID obtained
      }
      delay(50);
    }

    /* ---- 2) first scan + duplicate check ----- */
    lcd.clear(); lcd.setCursor(0, 0); lcd.print("Put finger #1");
    while (finger.getImage() != FINGERPRINT_OK) {
      if (pressed(BTN_CNC)) goto exit_enrol;
      delay(200);
    }
    if (finger.image2Tz(1) != FINGERPRINT_OK) {
      SerialBT.println("ERR: image2Tz(1)");
      lcd.clear(); lcd.setCursor(0,1); lcd.print("Scan failed");
      goto show_result;                           // jump to OK/CANCEL dialogue
    }

    if (finger.fingerFastSearch() == FINGERPRINT_OK) {
      SerialBT.printf("DUPLICATE – already ID %d\n", finger.fingerID);
      lcd.clear(); lcd.setCursor(0,0); lcd.print("Duplicate!");
      lcd.setCursor(0,1); lcd.print("ID="); lcd.print(finger.fingerID);
      goto show_result;
    }

    /* ---- 3) second scan ----------------------- */
    lcd.clear(); lcd.setCursor(0, 0); lcd.print("Remove & again");
    delay(1000);
    lcd.clear(); lcd.setCursor(0, 0); lcd.print("Put finger #2");
    while (finger.getImage() != FINGERPRINT_OK) {
      if (pressed(BTN_CNC)) goto exit_enrol;
      delay(200);
    }
    if (finger.image2Tz(2) != FINGERPRINT_OK) {
      SerialBT.println("ERR: image2Tz(2)");
      lcd.clear(); lcd.setCursor(0,1); lcd.print("Scan failed");
      goto show_result;
    }

    /* ---- 4) create & store model -------------- */
    lcd.clear(); lcd.setCursor(0, 0); lcd.print("Creating model");
    if (finger.createModel() != FINGERPRINT_OK) {
      SerialBT.println("ERR: createModel");
      lcd.clear(); lcd.setCursor(0,1); lcd.print("Model failed");
      goto show_result;
    }

    lcd.clear(); lcd.setCursor(0, 0); lcd.print("Saving ...");
    if (finger.storeModel(newID) == FINGERPRINT_OK) {
      SerialBT.println("DONE");
      lcd.clear(); lcd.setCursor(0,1); lcd.print("Done");
    } else {
      SerialBT.println("ERR: storeModel");
      lcd.clear(); lcd.setCursor(0,1); lcd.print("Save failed");
    }

show_result:
    /* ---- 5) wait for operator choice ---------- */
    lcd.setCursor(0, 3); lcd.print("OK=next  CNCL=exit");
    while (true) {
      if (pressed(BTN_OK))  {          // take another fingerprint
        while (pressed(BTN_OK)) delay(10);   // wait for release
        break;                       // break "choice loop" → outer while repeats
      }
      if (pressed(BTN_CNC)) goto exit_enrol;
      delay(50);
    }
  } // end while(true)  (next fingerprint)

exit_enrol:
  SerialBT.println("Leaving FP-menu");
  while (pressed(BTN_CNC)) delay(10);   // ensure button released
  drawMainMenu();
}
// 3) Voting flow
/* ─────────────────────────────────────────────────────────────
   Helper : draw 1…8 candidates in a two–column layout

   LCD: 20×4
   left column  starts at col 0
   right column starts at col 10
   Format per entry :  " >1.Name "
   (arrow only at the currently selected item)
   Name is truncated to 6-7 characters to fit in the 10-char field.
   ──────────────────────────────────────────────────────────── */
void drawCandidateGrid(int sel)
{
  lcd.clear();
  for (int row = 0; row < 4; ++row) {

    /* left column (idx = row) */
    int idxL = row;
    if (idxL < candCount) {
      lcd.setCursor(0, row);
      lcd.print(sel == idxL ? '>' : ' ');
      lcd.print(idxL + 1);          // numbering starts with 1
      lcd.print('.');
      String name = candidates[idxL];
      if (name.length() > 6) name = name.substring(0, 6);   // keep it short
      lcd.print(name);
    }

    /* right column (idx = row + 4) */
    int idxR = row + 4;
    if (idxR < candCount) {
      lcd.setCursor(10, row);       // start of right column
      lcd.print(sel == idxR ? '>' : ' ');
      lcd.print(idxR + 1);
      lcd.print('.');
      String name = candidates[idxR];
      if (name.length() > 6) name = name.substring(0, 6);
      lcd.print(name);
    }
  }
}


/* ─────────────────────────────────────────────────────────────
   3)  GO-FOR-VOTE   (improved)
   • if fingerprint already voted  → message “Already Voted”
   • else shows a 2-column grid (1-4 left, 5-8 right)
     navigation :  UP / DOWN  = ±1
                   LEFT / RIGHT = −4 / +4
     OK     = cast vote
     CANCEL = abort
   ──────────────────────────────────────────────────────────── */
void goForVote()
{
  /* sanity checks ------------------------------------------------ */
  if (candCount == 0) {
    lcd.clear(); lcd.setCursor(0, 1); lcd.print("No candidates");
    delay(1000); drawMainMenu(); return;
  }
  if (votingClosed) {
    lcd.clear(); lcd.setCursor(2, 1); lcd.print("Voting Closed!");
    delay(1000); drawMainMenu(); return;
  }

  /* fingerprint authentication ---------------------------------- */
  int fid = authenticate();
  if (fid < 0) { drawMainMenu(); return; }

  if (fid <= MAX_FID && votedIDs[fid]) {
    lcd.clear(); lcd.setCursor(0, 1); lcd.print("Already Voted");
    delay(1000); drawMainMenu(); return;
  }

  /* candidate selection ----------------------------------------- */
  int sel = 0;                        // current highlighted candidate
  drawCandidateGrid(sel);

  while (true) {

    /* navigation ------------------------------------------------ */
    if (pressed(BTN_UP) && sel > 0) {
      sel--;       drawCandidateGrid(sel);  delay(200);
    }
    if (pressed(BTN_DWN) && sel < candCount - 1) {
      sel++;       drawCandidateGrid(sel);  delay(200);
    }
    if (pressed(BTN_LFT) && sel >= 4) {     // jump to left column
      sel -= 4;   drawCandidateGrid(sel);   delay(200);
    }
    if (pressed(BTN_RHT) && sel + 4 < candCount) {  // right col
      sel += 4;   drawCandidateGrid(sel);   delay(200);
    }

    /* vote cast ------------------------------------------------- */
    if (pressed(BTN_OK)) {
      votesTally[sel]++;
      if (fid <= MAX_FID) votedIDs[fid] = true;

      /* flash LED three times */
      for (int i = 0; i < 3; ++i) {
        digitalWrite(LED_PIN, HIGH); delay(150);
        digitalWrite(LED_PIN, LOW);  delay(150);
      }

      lcd.clear();
      lcd.setCursor(0, 1); lcd.print("Voted for:");
      lcd.setCursor(0, 2); lcd.print(candidates[sel]);
      delay(1000); drawMainMenu(); return;
    }

    /* abort ----------------------------------------------------- */
    if (pressed(BTN_CNC)) {
      lcd.clear(); lcd.setCursor(0, 1); lcd.print("Vote canceled");
      delay(800); drawMainMenu(); return;
    }
  }
}
// 4) Display winner
// ─────────────────────────────────────────────────────────────
// 4) SHOW WINNER   (password-protected + closes the election)
// ─────────────────────────────────────────────────────────────
// ─────────────────────────────────────────────────────────────
// 4) SHOW WINNER   (re-worked)
//    • First call    → asks admin PW, closes voting, shows result
//    • Later calls   → shows the same result immediately
//    • Screen stays until CANCEL (or OK) is pressed
// ─────────────────────────────────────────────────────────────
void showWinner()
{
    /* ensure the key that opened this menu is released first */
  while (pressed(BTN_OK) || pressed(BTN_CNC)) delay(10);
  /* ───────── step 1 : if not yet closed → ask admin password ───────── */
  if (!votingClosed) {

    SerialBT.printf("ENTER ADMIN PASSWORD to finish vote (%d tries).\n"
                    "Type CANCEL or press button to abort.\n", maxTries);
    lcd.clear(); lcd.setCursor(0, 1); lcd.print("Admin PW ?");

    String in;  uint8_t tries = 0;
    while (true) {
      if (pressed(BTN_CNC)) {             // HW cancel
        SerialBT.println("CANCELLED (button)");
        waitForCancel(); drawMainMenu();  return;
      }
      if (SerialBT.available()) {
        in = SerialBT.readStringUntil('\n'); in.trim();
        if (in.equalsIgnoreCase("CANCEL")) {
          SerialBT.println("CANCELLED");  drawMainMenu(); return;
        }
        if (in == adminPass) {            // success
          SerialBT.println("OK — Vote finished");
          break;
        }
        if (++tries >= maxTries) {
          SerialBT.println("TOO MANY FAILS — ABORT");
          lcd.clear(); lcd.setCursor(0, 1); lcd.print("Too many fails");
          delay(1000); drawMainMenu(); return;
        }
        SerialBT.printf("WRONG (%d/%d)\n", tries, maxTries);
      }
      delay(50);
    }
    votingClosed = true;                  // lock election from now on
  }

  /* ───────── step 2 : compute current winner (can be called many times) ───── */
  if (candCount == 0) {                   // safety
    lcd.clear(); lcd.setCursor(0, 1); lcd.print("No candidates");
    delay(1000); drawMainMenu(); return;
  }

  int best = 0;
  for (int i = 1; i < candCount; ++i)
    if (votesTally[i] > votesTally[best]) best = i;

  /* ───────── step 3 : show result ────────────────────────────────────── */
  lcd.clear();
  lcd.setCursor(2, 0); lcd.print("VOTING CLOSED");
  lcd.setCursor(0, 1); lcd.print("Winner:");
  lcd.setCursor(0, 2);
  lcd.print(String(best + 1) + "." + candidates[best]);
  lcd.setCursor(0, 3);
  lcd.print("Votes="); lcd.print(votesTally[best]);
  lcd.setCursor(15, 3); lcd.print("<CNC");

  /* also send info over Bluetooth (every time) */
  SerialBT.printf("Winner: %d.%s with %d vote(s)\n",
                  best + 1, candidates[best].c_str(), votesTally[best]);

  /* ───────── step 4 : wait until operator leaves via CANCEL or OK ───── */
  while (!pressed(BTN_CNC) && !pressed(BTN_OK)) delay(20);
  while ( pressed(BTN_CNC) || pressed(BTN_OK)) delay(20);  // wait release
  drawMainMenu();
}
/* print a text centred on a 20-column LCD row */
void centerText(uint8_t row, const char *txt)
{
  int col = (20 - (int)strlen(txt)) / 2;   // starting column
  if (col < 0) col = 0;                    // clip if text is >20 chars
  lcd.setCursor(col, row);
  lcd.print(txt);
}
// ── SETUP ─────────────────────────────────────────────────────
void setup() {
  // buttons / LED
  pinMode(BTN_OK,  INPUT_PULLUP);
  pinMode(BTN_CNC, INPUT_PULLUP);
  pinMode(BTN_UP,  INPUT_PULLUP);
  pinMode(BTN_DWN, INPUT_PULLUP);
  pinMode(BTN_LFT, INPUT_PULLUP);   // ← added back
  pinMode(BTN_RHT, INPUT_PULLUP);   // ← added back
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  // LCD
  Wire.begin();
  lcd.init(); lcd.backlight(); lcd.clear();

  // Fingerprint
  Serial2.begin(57600, SERIAL_8N1, FINGER_RX_PIN, FINGER_TX_PIN);
  finger.begin(57600);
  if (!finger.verifyPassword()) {
    lcd.setCursor(0, 1); lcd.print("Finger ERR");
    while (true) delay(1000);
  }

  // Bluetooth admin login
  SerialBT.begin("ESP32Votes");
  int tries = 0; bool auth = false;
  while (tries < maxTries) {
    SerialBT.println("\nENTER ADMIN PASS:");
    lcd.clear();
    centerText(1, "Admin login");   // centred on row 1

    String at;
    while (at.length() == 0) {
      if (SerialBT.available()) {
        at = SerialBT.readStringUntil('\n'); at.trim();
      }
      delay(50);
    }
    if (at == adminPass) {
      auth = true;
      SerialBT.println("OK");
      break;
    }
    SerialBT.println("WRONG"); tries++;
  }
  if (!auth) {
    lcd.clear(); lcd.setCursor(2, 1);
    lcd.print("LOCKED OUT");
    while (true) delay(500);
  }

  // ► load candidates from flash
  preferences.begin("vote", true);
  candCount = preferences.getUChar("candCount", 0);
  if (candCount > 8) candCount = 0;      // sanity check
  for (int i = 0; i < candCount; ++i) {
    candidates[i] = preferences.getString(
                      ("cand" + String(i)).c_str(), "NoName");
    votesTally[i] = 0;   // fresh vote counts each power-up
  }
  preferences.end();
/* ── Simple Wi-Fi AP + web page ─────────────────────────── */
WiFi.mode(WIFI_AP);
WiFi.softAP(WIFI_SSID, WIFI_PASSWORD);

SerialBT.printf("Wi-Fi AP ready.  SSID: %s  IP: %s\n",
                WIFI_SSID,
                WiFi.softAPIP().toString().c_str());

server.on("/", HTTP_GET, [](){
  server.send(200, "text/html", buildMenuPage());
});
server.begin();

/* ───────────────────────────────────────────────────────────── */

/* existing “welcome” lines remain unchanged */
  lcd.clear(); lcd.setCursor(2, 1); lcd.print("Welcome Admin");
  delay(1000);
  drawMainMenu();
}

// ── LOOP ─────────────────────────────────────────────────────
void loop() {

   server.handleClient();          // ← must be called often
  // Check Bluetooth commands
  if (SerialBT.available()) {
    String cmd = SerialBT.readStringUntil('\n'); cmd.trim();
    if (cmd.equalsIgnoreCase("FINISH")) {
      votingClosed = true;
      SerialBT.println("VOTING CLOSED");
      showWinner();
    }
  }

  // main-menu navigation
  if (pressed(BTN_UP)  && menuIndex > 0) {
    menuIndex--;
    drawMainMenu();
    delay(200);
  }
  if (pressed(BTN_DWN) && menuIndex < 3) {
    menuIndex++;
    drawMainMenu();
    delay(200);
  }
  if (pressed(BTN_OK)) {
    switch (menuIndex) {
      case 0: saveCandidates();    break;
      case 1: enrollFingerprint(); break;
      case 2: goForVote();         break;
      case 3: showWinner();        break;
    }
    delay(200);
  }
  // CANCEL on main menu does nothing
}
