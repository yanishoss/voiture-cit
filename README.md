# Voiture en Python
> Un défi à réaliser, coder une voiture avec Arduino et Python

## Introduction
Pour coder une voiture avec Arduino et Python, il faut coder **une partie en C** car Python ne **peut pas être lancé sur Arduino.**   
On va alors créer un programme en C mis sur la carte **qui communiquera avec un script Python** mis sur un ordinateur ou un Raspberry Pi.   

## La partie C
Le programme en C s'occupe **d'écouter sur le port série** et d'effectuer les commandes que le script Python lui demande. C'est une **interface de programmation** qu'on appelle communément API (Application Programming Interface).

```c
/*
Ce programme est écrit en anglais pour des raisons de style, malgré tout il est commenté en francais.
Il a été écrit en C pour la plateforme Arduino.

Il est fait pour fonctionner de pair avec le module Python associé.
En effet, en branchant par USB votre carte Arduino sur un ordinateur équipé de Python ou sur un Raspberry Pi,
vous pourrez contrôler votre voiture grâce à Python qui est un langage plus simple que le C, avec un plus grand niveau d'abstraction.

Ce programme a été écrit par Yanis HEDJEM.
*/

// La pin sur laquelle le moteur est branché.
// La pin doit être une pin analog.
const int motor_pin = 9;

// La pin sur laquelle le capteur de sensibilité est branchée.
// La pin doit être une pin analog PWM.
const int photocell_pin = 8;

// Le seuil de luminosité à partir duquel le capteur de luminosité s'active.
// Le seuil va de 0 (moins lumineux, correspond à la nuit) à 800 (plus lumineux, correspond à un flash).
const int photocell_threshold = 600;

// La pin sur laquelle la première borne du transistor est branchée.
// La pin doit être une pin digital.
const int in1_transistor_pin = 2;

// La pin sur laquelle la deuxième borne du transistor est branchée.
// La pin doit être une pin digital.
const int in2_transistor_pin = 2;

enum Command {
  GO_FORWARD = 1,
  GO_BACKWARD = 2,
  BREAK_WHEEL = 3
};

void setup() {
  // Met en marche le port série.
  Serial.begin(9600);

  pinMode(motor_pin, OUTPUT);
  pinMode(photocell_pin, INPUT);
  pinMode(in1_transistor_pin, OUTPUT);
  pinMode(in2_transistor_pin, OUTPUT);
}

void loop() {
  // Envoie sur le port série l'état du capteur de luminosité.
  if (Serial.availableForWrite()) {
    Serial.write(is_light_on());
  }
  
  dispatch();
}

// Permet de savoir si le capteur est activé.
bool is_light_on() {
  return analogRead(photocell_pin) >= photocell_threshold;
}

// Permet à partir du port série de savoir quel commande exécuter (avancer, freiner, reculer, ...).
void dispatch() {
  while (Serial.available()) {
    char command;
    Serial.readBytes(&command, 1);

    char speed_percentage;

    switch (command) {
     case GO_FORWARD: 
      Serial.readBytes(&speed_percentage, 1);
      go_forward(speed_percentage);
      break;
     case GO_BACKWARD:
      Serial.readBytes(&speed_percentage, 1);
      go_backward(speed_percentage);
      break;
     case BREAK_WHEEL:
      break_wheel();
      break;
     default:
      continue;
    }
  }
}

// Permet d'accélèrer en avant.
void go_forward(int speed_percentage) {
  // Met la marche avant grâce au transistor (inverse les polarités).
  digitalWrite(in1_transistor_pin, LOW);
  digitalWrite(in2_transistor_pin, HIGH);

  // Accélère le moteur.
  analogWrite(motor_pin, (speed_percentage/100)*255);
}

// Permet d'accélèrer en arrière.
void go_backward(int speed_percentage) {
  // Met la marche arrière grâce au transistor (inverse les polarités).
  digitalWrite(in1_transistor_pin, HIGH);
  digitalWrite(in2_transistor_pin, LOW);

  // Accélère le moteur.
  analogWrite(motor_pin, (speed_percentage/100)*255);
}

// Permet de freiner.
void break_wheel() {
  // Coupe le courant au moteur grâce au transistor.
  digitalWrite(in1_transistor_pin, HIGH);
  digitalWrite(in2_transistor_pin, HIGH);

  // Garde le freinage pendant au moins 1 secondes.
  delay(1000);
}

```

## La partie Python
Nous allons maintenant écrire un module Python qui permettra à qui le veut d'utiliser notre voiture simplement (avec des commandes simples, avec **un plus haut niveau d'abstraction**).

```python
# Importe le module Python pour manipuler les ports séries.
import serial

GO_FORWARD = 1
GO_BACKWARD = 2
BREAK_WHEEL = 3

conn = None

light_on = False

# Initialise la communication avec l'Arduino.
# Veillez bien à mettre le port USB correct (COM1, COM2, ...).
def start(port):
  with serial.Serial(port=port, baudrate=9600, timeout=1, writeTimeout=1) as connection:
    conn = connection

# Accélère en avant.
def go_forward(speed):
  if conn == None:
    return 
  conn.write(bytes(GO_FORWARD))
  conn.write(bytes(speed))
  
# Accélère en arrière.
def go_backward(speed):
   if conn == None:
    return 
  conn.write(bytes(GO_BACKWARD))
  conn.write(bytes(speed))

# Freine la voiture.
def break_wheel():
   if conn == None:
    return 
  conn.write(bytes(BREAK_WHEEL))

# Donne l'état du capteur de luminosité sous forme de boolean.
def is_light_on():
  while conn.in_waiting > 0:
    on = conn.read(1)
    light_on = on == 1
  return light_on
```

Et voici !   
Reste plus qu'à adapter le code aux branchements sur la carte Arduino et à jouer :)   

**22/20 ou pas ?**   
