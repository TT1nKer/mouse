# Phase 2: Basic Implementation - Detailed Step-by-Step Guide
## nRF52840 Wireless Mouse Development

---

## 📋 **Phase 2 Overview**
Duration: 3-5 weeks  
Goal: Create a working prototype with basic mouse functionality

---

## 🎯 **Section 2.1: Hello World & Basic I/O**

### **Step 1: Set Up Your First Project**

#### **1.1 Create New Project**
1. **Open nRF Connect for Desktop**
   - Launch `nRF Connect for Desktop`
   - Click on `Toolchain Manager`
   - Select your installed SDK version (e.g., v2.4.0)
   - Click `Open VS Code`

2. **Create Application**
   ```bash
   # In VS Code terminal (Ctrl+`)
   cd D:\mouse
   west init -m https://github.com/nrfconnect/sdk-nrf --mr v2.4.0 nrf-mouse-project
   cd nrf-mouse-project
   west update
   ```

3. **Create Your Application Folder**
   ```bash
   mkdir applications\mouse_basic
   cd applications\mouse_basic
   ```

#### **1.2 First Blink LED Example**

1. **Create `prj.conf` file**
   ```ini
   # Enable GPIO
   CONFIG_GPIO=y
   
   # Enable logging
   CONFIG_LOG=y
   CONFIG_LOG_DEFAULT_LEVEL=3
   CONFIG_CONSOLE=y
   CONFIG_UART_CONSOLE=y
   
   # Enable button and LED libraries
   CONFIG_DK_LIBRARY=y
   ```

2. **Create `CMakeLists.txt`**
   ```cmake
   cmake_minimum_required(VERSION 3.20.0)
   
   find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
   project(mouse_basic)
   
   target_sources(app PRIVATE src/main.c)
   ```

3. **Create `src/main.c`**
   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/device.h>
   #include <zephyr/drivers/gpio.h>
   #include <zephyr/logging/log.h>
   #include <dk_buttons_and_leds.h>
   
   LOG_MODULE_REGISTER(mouse_basic, LOG_LEVEL_INF);
   
   #define LED_BLINK_INTERVAL_MS 1000
   
   void main(void)
   {
       int err;
       
       LOG_INF("Starting nRF52840 Mouse Basic Example");
       
       // Initialize DK buttons and LEDs
       err = dk_leds_init();
       if (err) {
           LOG_ERR("Cannot init LEDs (err: %d)", err);
           return;
       }
       
       LOG_INF("LED initialized successfully");
       
       // Main loop - blink LED
       while (1) {
           dk_set_led_on(DK_LED1);
           k_sleep(K_MSEC(LED_BLINK_INTERVAL_MS / 2));
           
           dk_set_led_off(DK_LED1);
           k_sleep(K_MSEC(LED_BLINK_INTERVAL_MS / 2));
           
           LOG_INF("LED blink cycle completed");
       }
   }
   ```

4. **Build and Flash**
   ```bash
   # In VS Code terminal
   west build -b nrf52840dk_nrf52840
   west flash
   ```

5. **Verify Output**
   - LED1 should blink every second
   - Open serial terminal: `nRF Connect for Desktop` → `Serial Terminal`
   - Select your device, baud rate 115200
   - You should see log messages

#### **1.3 Add Button Input Handling**

1. **Update `src/main.c` - Add Button Support**
   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/device.h>
   #include <zephyr/drivers/gpio.h>
   #include <zephyr/logging/log.h>
   #include <dk_buttons_and_leds.h>
   
   LOG_MODULE_REGISTER(mouse_basic, LOG_LEVEL_INF);
   
   #define LED_BLINK_INTERVAL_MS 1000
   
   // Button callback function
   static void button_changed(uint32_t button_state, uint32_t has_changed)
   {
       if (has_changed & DK_BTN1_MSK) {
           if (button_state & DK_BTN1_MSK) {
               LOG_INF("Button 1 pressed");
               dk_set_led_on(DK_LED2);  // Turn on LED2 when button pressed
           } else {
               LOG_INF("Button 1 released");
               dk_set_led_off(DK_LED2); // Turn off LED2 when button released
           }
       }
       
       if (has_changed & DK_BTN2_MSK) {
           if (button_state & DK_BTN2_MSK) {
               LOG_INF("Button 2 pressed");
               dk_set_led_on(DK_LED3);
           } else {
               LOG_INF("Button 2 released");
               dk_set_led_off(DK_LED3);
           }
       }
   }
   
   void main(void)
   {
       int err;
       
       LOG_INF("Starting nRF52840 Mouse Basic Example with Buttons");
       
       // Initialize DK buttons and LEDs
       err = dk_leds_init();
       if (err) {
           LOG_ERR("Cannot init LEDs (err: %d)", err);
           return;
       }
       
       err = dk_buttons_init(button_changed);
       if (err) {
           LOG_ERR("Cannot init buttons (err: %d)", err);
           return;
       }
       
       LOG_INF("LEDs and buttons initialized successfully");
       
       // Main loop - blink LED1
       while (1) {
           dk_set_led_on(DK_LED1);
           k_sleep(K_MSEC(LED_BLINK_INTERVAL_MS / 2));
           
           dk_set_led_off(DK_LED1);
           k_sleep(K_MSEC(LED_BLINK_INTERVAL_MS / 2));
       }
   }
   ```

2. **Build and Test**
   ```bash
   west build -b nrf52840dk_nrf52840
   west flash
   ```

3. **Test Functionality**
   - Press Button 1 → LED2 should light up
   - Press Button 2 → LED3 should light up
   - Check serial output for button press/release messages

#### **1.4 Implement Button Debouncing**

1. **Create `src/button_handler.c`**
   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/logging/log.h>
   #include "button_handler.h"
   
   LOG_MODULE_REGISTER(button_handler, LOG_LEVEL_DBG);
   
   #define DEBOUNCE_TIME_MS 50
   
   static struct k_work_delayable button_work[MAX_BUTTONS];
   static button_callback_t button_callbacks[MAX_BUTTONS];
   static uint32_t button_states = 0;
   static uint32_t last_button_states = 0;
   
   static void button_work_handler(struct k_work *work)
   {
       struct k_work_delayable *delayable_work = 
           CONTAINER_OF(work, struct k_work_delayable, work);
       
       int button_id = delayable_work - button_work;
       
       if (button_id < 0 || button_id >= MAX_BUTTONS) {
           return;
       }
       
       uint32_t current_state = button_states & BIT(button_id);
       uint32_t last_state = last_button_states & BIT(button_id);
       
       if (current_state != last_state) {
           if (button_callbacks[button_id]) {
               button_callbacks[button_id](button_id, current_state ? true : false);
           }
           
           if (current_state) {
               last_button_states |= BIT(button_id);
           } else {
               last_button_states &= ~BIT(button_id);
           }
           
           LOG_DBG("Button %d state changed to %s", 
                   button_id, current_state ? "pressed" : "released");
       }
   }
   
   void button_handler_init(void)
   {
       for (int i = 0; i < MAX_BUTTONS; i++) {
           k_work_init_delayable(&button_work[i], button_work_handler);
           button_callbacks[i] = NULL;
       }
       
       LOG_INF("Button handler initialized");
   }
   
   void button_handler_register_callback(int button_id, button_callback_t callback)
   {
       if (button_id >= 0 && button_id < MAX_BUTTONS) {
           button_callbacks[button_id] = callback;
           LOG_DBG("Callback registered for button %d", button_id);
       }
   }
   
   void button_handler_update_state(int button_id, bool pressed)
   {
       if (button_id < 0 || button_id >= MAX_BUTTONS) {
           return;
       }
       
       if (pressed) {
           button_states |= BIT(button_id);
       } else {
           button_states &= ~BIT(button_id);
       }
       
       // Schedule debounced work
       k_work_reschedule(&button_work[button_id], K_MSEC(DEBOUNCE_TIME_MS));
   }
   ```

2. **Create `src/button_handler.h`**
   ```c
   #ifndef BUTTON_HANDLER_H
   #define BUTTON_HANDLER_H
   
   #include <zephyr/types.h>
   
   #define MAX_BUTTONS 8
   
   typedef void (*button_callback_t)(int button_id, bool pressed);
   
   void button_handler_init(void);
   void button_handler_register_callback(int button_id, button_callback_t callback);
   void button_handler_update_state(int button_id, bool pressed);
   
   #endif /* BUTTON_HANDLER_H */
   ```

3. **Update `CMakeLists.txt`**
   ```cmake
   cmake_minimum_required(VERSION 3.20.0)
   
   find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
   project(mouse_basic)
   
   target_sources(app PRIVATE 
       src/main.c
       src/button_handler.c
   )
   
   target_include_directories(app PRIVATE src)
   ```

---

## 🎯 **Section 2.2: Sensor Integration**

### **Step 2: Research and Select Optical Sensor**

#### **2.1 Sensor Research**

1. **Download Datasheets**
   - Go to PixArt website: `https://www.pixart.com/`
   - Download datasheets for:
     - PMW3360 (gaming sensor)
     - PMW3389 (high-end gaming)
     - PMW3325 (office sensor)

2. **Compare Specifications**
   Create a comparison table in Excel/Google Sheets:
   ```
   | Sensor   | Max DPI | Max Speed | Acceleration | Interface | Price |
   |----------|---------|-----------|--------------|-----------|-------|
   | PMW3360  | 12000   | 250 IPS   | 50G          | SPI       | $8-12 |
   | PMW3389  | 16000   | 400 IPS   | 50G          | SPI       | $12-18|
   | PMW3325  | 5000    | 100 IPS   | 20G          | SPI       | $4-8  |
   ```

3. **For This Tutorial: Use PMW3360**
   - Good balance of performance and availability
   - Well-documented
   - Commonly available on breakout boards

#### **2.2 Acquire PMW3360 Breakout Board**

1. **Purchase Options**
   - **Tindie**: Search "PMW3360 breakout"
   - **AliExpress**: Search "PMW3360 module"
   - **eBay**: Search "PMW3360 sensor board"

2. **Alternative: Create Your Own Board**
   - Download PMW3360 reference design from PixArt
   - Use KiCad to create breakout board
   - Order from JLCPCB or PCBWay

#### **2.3 Wire PMW3360 to nRF52840DK**

1. **Connection Diagram**
   ```
   PMW3360 Breakout    nRF52840DK
   ================    ==========
   VCC (3.3V)     →    VDD (P21)
   GND            →    GND (P22)
   SCLK           →    P0.03 (SPI_SCK)
   MOSI           →    P0.04 (SPI_MOSI)
   MISO           →    P0.28 (SPI_MISO)
   NCS            →    P0.29 (SPI_CS)
   MOTION         →    P0.30 (GPIO_INT)
   ```

2. **Physical Wiring**
   - Use breadboard and jumper wires
   - Double-check connections with multimeter
   - Ensure 3.3V power supply (NOT 5V!)

### **Step 3: Implement SPI Communication**

#### **3.1 Update Project Configuration**

1. **Update `prj.conf`**
   ```ini
   # Previous configs...
   CONFIG_GPIO=y
   CONFIG_LOG=y
   CONFIG_LOG_DEFAULT_LEVEL=3
   CONFIG_CONSOLE=y
   CONFIG_UART_CONSOLE=y
   CONFIG_DK_LIBRARY=y
   
   # Add SPI support
   CONFIG_SPI=y
   CONFIG_SPI_NRFX=y
   
   # Add sensor support
   CONFIG_SENSOR=y
   ```

2. **Create Device Tree Overlay**
   Create `nrf52840dk_nrf52840.overlay`:
   ```dts
   &spi1 {
       compatible = "nordic,nrf-spi";
       status = "okay";
       
       pinctrl-0 = <&spi1_default>;
       pinctrl-1 = <&spi1_sleep>;
       pinctrl-names = "default", "sleep";
       
       cs-gpios = <&gpio0 29 GPIO_ACTIVE_LOW>;
       
       pmw3360: pmw3360@0 {
           compatible = "pixart,pmw3360";
           reg = <0>;
           spi-max-frequency = <2000000>;
           motion-gpios = <&gpio0 30 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
       };
   };
   
   &pinctrl {
       spi1_default: spi1_default {
           group1 {
               psels = <NRF_PSEL(SPIM_SCK, 0, 3)>,
                       <NRF_PSEL(SPIM_MOSI, 0, 4)>,
                       <NRF_PSEL(SPIM_MISO, 0, 28)>;
           };
       };
       
       spi1_sleep: spi1_sleep {
           group1 {
               psels = <NRF_PSEL(SPIM_SCK, 0, 3)>,
                       <NRF_PSEL(SPIM_MOSI, 0, 4)>,
                       <NRF_PSEL(SPIM_MISO, 0, 28)>;
               low-power-enable;
           };
       };
   };
   ```

#### **3.2 Create PMW3360 Driver**

1. **Create `src/pmw3360.h`**
   ```c
   #ifndef PMW3360_H
   #define PMW3360_H
   
   #include <zephyr/device.h>
   #include <zephyr/drivers/spi.h>
   #include <zephyr/drivers/gpio.h>
   
   // PMW3360 Register Addresses
   #define PMW3360_REG_PRODUCT_ID          0x00
   #define PMW3360_REG_REVISION_ID         0x01
   #define PMW3360_REG_MOTION              0x02
   #define PMW3360_REG_DELTA_X_L           0x03
   #define PMW3360_REG_DELTA_X_H           0x04
   #define PMW3360_REG_DELTA_Y_L           0x05
   #define PMW3360_REG_DELTA_Y_H           0x06
   #define PMW3360_REG_SQUAL               0x07
   #define PMW3360_REG_CONFIG2             0x10
   #define PMW3360_REG_ANGLE_TUNE          0x11
   #define PMW3360_REG_FRAME_CAPTURE       0x12
   #define PMW3360_REG_SROM_ENABLE         0x13
   #define PMW3360_REG_RUN_DOWNSHIFT       0x14
   #define PMW3360_REG_REST1_RATE_LOWER    0x15
   #define PMW3360_REG_REST1_RATE_UPPER    0x16
   #define PMW3360_REG_REST1_DOWNSHIFT     0x17
   #define PMW3360_REG_REST2_RATE_LOWER    0x18
   #define PMW3360_REG_REST2_RATE_UPPER    0x19
   #define PMW3360_REG_REST2_DOWNSHIFT     0x1A
   #define PMW3360_REG_REST3_RATE_LOWER    0x1B
   #define PMW3360_REG_REST3_RATE_UPPER    0x1C
   #define PMW3360_REG_OBSERVATION         0x24
   #define PMW3360_REG_DATA_OUT_LOWER      0x25
   #define PMW3360_REG_DATA_OUT_UPPER      0x26
   #define PMW3360_REG_RAW_DATA_SUM        0x29
   #define PMW3360_REG_SROM_ID             0x2A
   #define PMW3360_REG_MIN_SQ_RUN          0x2B
   #define PMW3360_REG_RAW_DATA_THRESHOLD  0x2C
   #define PMW3360_REG_CONFIG5             0x2F
   #define PMW3360_REG_POWER_UP_RESET      0x3A
   #define PMW3360_REG_SHUTDOWN            0x3B
   #define PMW3360_REG_INVERSE_PRODUCT_ID  0x3F
   #define PMW3360_REG_LIFTCUTOFF_TUNE3    0x41
   #define PMW3360_REG_ANGLE_SNAP          0x42
   #define PMW3360_REG_LIFTCUTOFF_TUNE1    0x4A
   #define PMW3360_REG_MOTION_BURST        0x50
   #define PMW3360_REG_LIFTCUTOFF_TUNE_TIMEOUT 0x58
   #define PMW3360_REG_LIFTCUTOFF_TUNE_MIN_LENGTH 0x5A
   #define PMW3360_REG_SROM_LOAD_BURST     0x62
   #define PMW3360_REG_LIFT_CONFIG         0x63
   #define PMW3360_REG_RAW_DATA_BURST      0x64
   #define PMW3360_REG_LIFTCUTOFF_TUNE2    0x65
   
   // PMW3360 Values
   #define PMW3360_PRODUCT_ID              0x42
   #define PMW3360_INVERSE_PRODUCT_ID      0xBD
   
   struct pmw3360_motion_data {
       int16_t delta_x;
       int16_t delta_y;
       uint8_t motion;
       uint8_t surface_quality;
   };
   
   struct pmw3360_config {
       struct spi_dt_spec spi;
       struct gpio_dt_spec motion_gpio;
   };
   
   int pmw3360_init(const struct device *dev);
   int pmw3360_read_motion(const struct device *dev, struct pmw3360_motion_data *data);
   int pmw3360_set_cpi(const struct device *dev, uint16_t cpi);
   int pmw3360_power_down(const struct device *dev);
   int pmw3360_power_up(const struct device *dev);
   
   #endif /* PMW3360_H */
   ```

2. **Create `src/pmw3360.c`**
   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/device.h>
   #include <zephyr/drivers/spi.h>
   #include <zephyr/drivers/gpio.h>
   #include <zephyr/logging/log.h>
   #include "pmw3360.h"
   
   LOG_MODULE_REGISTER(pmw3360, LOG_LEVEL_DBG);
   
   #define PMW3360_SPI_READ    0x00
   #define PMW3360_SPI_WRITE   0x80
   
   static int pmw3360_reg_read(const struct device *dev, uint8_t reg, uint8_t *data)
   {
       const struct pmw3360_config *config = dev->config;
       
       uint8_t tx_buf[2] = {reg & 0x7F, 0x00}; // Read bit = 0
       uint8_t rx_buf[2];
       
       struct spi_buf tx_spi_buf = {
           .buf = tx_buf,
           .len = sizeof(tx_buf)
       };
       
       struct spi_buf rx_spi_buf = {
           .buf = rx_buf,
           .len = sizeof(rx_buf)
       };
       
       struct spi_buf_set tx_spi_buf_set = {
           .buffers = &tx_spi_buf,
           .count = 1
       };
       
       struct spi_buf_set rx_spi_buf_set = {
           .buffers = &rx_spi_buf,
           .count = 1
       };
       
       int ret = spi_transceive_dt(&config->spi, &tx_spi_buf_set, &rx_spi_buf_set);
       if (ret < 0) {
           LOG_ERR("SPI read failed: %d", ret);
           return ret;
       }
       
       *data = rx_buf[1];
       
       // PMW3360 requires delay after read
       k_sleep(K_USEC(35));
       
       return 0;
   }
   
   static int pmw3360_reg_write(const struct device *dev, uint8_t reg, uint8_t data)
   {
       const struct pmw3360_config *config = dev->config;
       
       uint8_t tx_buf[2] = {reg | 0x80, data}; // Write bit = 1
       
       struct spi_buf tx_spi_buf = {
           .buf = tx_buf,
           .len = sizeof(tx_buf)
       };
       
       struct spi_buf_set tx_spi_buf_set = {
           .buffers = &tx_spi_buf,
           .count = 1
       };
       
       int ret = spi_write_dt(&config->spi, &tx_spi_buf_set);
       if (ret < 0) {
           LOG_ERR("SPI write failed: %d", ret);
           return ret;
       }
       
       // PMW3360 requires delay after write
       k_sleep(K_USEC(35));
       
       return 0;
   }
   
   int pmw3360_init(const struct device *dev)
   {
       const struct pmw3360_config *config = dev->config;
       uint8_t product_id, inverse_product_id;
       int ret;
       
       LOG_INF("Initializing PMW3360 sensor");
       
       if (!spi_is_ready_dt(&config->spi)) {
           LOG_ERR("SPI device not ready");
           return -ENODEV;
       }
       
       if (!gpio_is_ready_dt(&config->motion_gpio)) {
           LOG_ERR("Motion GPIO not ready");
           return -ENODEV;
       }
       
       // Configure motion pin as input
       ret = gpio_pin_configure_dt(&config->motion_gpio, GPIO_INPUT);
       if (ret < 0) {
           LOG_ERR("Cannot configure motion GPIO: %d", ret);
           return ret;
       }
       
       // Power up reset
       ret = pmw3360_reg_write(dev, PMW3360_REG_POWER_UP_RESET, 0x5A);
       if (ret < 0) {
           return ret;
       }
       
       k_sleep(K_MSEC(50));
       
       // Read product ID
       ret = pmw3360_reg_read(dev, PMW3360_REG_PRODUCT_ID, &product_id);
       if (ret < 0) {
           return ret;
       }
       
       ret = pmw3360_reg_read(dev, PMW3360_REG_INVERSE_PRODUCT_ID, &inverse_product_id);
       if (ret < 0) {
           return ret;
       }
       
       LOG_INF("Product ID: 0x%02X, Inverse: 0x%02X", product_id, inverse_product_id);
       
       if (product_id != PMW3360_PRODUCT_ID || inverse_product_id != PMW3360_INVERSE_PRODUCT_ID) {
           LOG_ERR("Invalid product ID");
           return -EINVAL;
       }
       
       LOG_INF("PMW3360 sensor initialized successfully");
       return 0;
   }
   
   int pmw3360_read_motion(const struct device *dev, struct pmw3360_motion_data *data)
   {
       uint8_t motion_reg;
       uint8_t delta_x_l, delta_x_h;
       uint8_t delta_y_l, delta_y_h;
       uint8_t squal;
       int ret;
       
       // Read motion register first
       ret = pmw3360_reg_read(dev, PMW3360_REG_MOTION, &motion_reg);
       if (ret < 0) {
           return ret;
       }
       
       data->motion = motion_reg;
       
       if (!(motion_reg & 0x80)) {
           // No motion detected
           data->delta_x = 0;
           data->delta_y = 0;
           data->surface_quality = 0;
           return 0;
       }
       
       // Read delta X (low byte first)
       ret = pmw3360_reg_read(dev, PMW3360_REG_DELTA_X_L, &delta_x_l);
       if (ret < 0) {
           return ret;
       }
       
       ret = pmw3360_reg_read(dev, PMW3360_REG_DELTA_X_H, &delta_x_h);
       if (ret < 0) {
           return ret;
       }
       
       // Read delta Y (low byte first)
       ret = pmw3360_reg_read(dev, PMW3360_REG_DELTA_Y_L, &delta_y_l);
       if (ret < 0) {
           return ret;
       }
       
       ret = pmw3360_reg_read(dev, PMW3360_REG_DELTA_Y_H, &delta_y_h);
       if (ret < 0) {
           return ret;
       }
       
       // Read surface quality
       ret = pmw3360_reg_read(dev, PMW3360_REG_SQUAL, &squal);
       if (ret < 0) {
           return ret;
       }
       
       // Combine bytes (two's complement)
       data->delta_x = (int16_t)((delta_x_h << 8) | delta_x_l);
       data->delta_y = (int16_t)((delta_y_h << 8) | delta_y_l);
       data->surface_quality = squal;
       
       return 0;
   }
   
   int pmw3360_set_cpi(const struct device *dev, uint16_t cpi)
   {
       // PMW3360 CPI is set in increments of 100
       // Value = (CPI / 100) - 1
       uint8_t cpi_value = (cpi / 100) - 1;
       
       LOG_INF("Setting CPI to %d (register value: 0x%02X)", cpi, cpi_value);
       
       return pmw3360_reg_write(dev, PMW3360_REG_CONFIG2, cpi_value);
   }
   
   int pmw3360_power_down(const struct device *dev)
   {
       return pmw3360_reg_write(dev, PMW3360_REG_SHUTDOWN, 0xB6);
   }
   
   int pmw3360_power_up(const struct device *dev)
   {
       return pmw3360_reg_write(dev, PMW3360_REG_POWER_UP_RESET, 0x5A);
   }
   ```

#### **3.3 Test Sensor Communication**

1. **Update `src/main.c` to Test Sensor**
   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/device.h>
   #include <zephyr/drivers/gpio.h>
   #include <zephyr/logging/log.h>
   #include <dk_buttons_and_leds.h>
   #include "pmw3360.h"
   #include "button_handler.h"
   
   LOG_MODULE_REGISTER(mouse_basic, LOG_LEVEL_INF);
   
   #define SENSOR_POLL_INTERVAL_MS 10
   
   // Get sensor device from device tree
   #define PMW3360_NODE DT_NODELABEL(pmw3360)
   static const struct device *sensor_dev = DEVICE_DT_GET(PMW3360_NODE);
   
   // Button callbacks
   static void left_button_callback(int button_id, bool pressed)
   {
       LOG_INF("Left mouse button %s", pressed ? "pressed" : "released");
       if (pressed) {
           dk_set_led_on(DK_LED2);
       } else {
           dk_set_led_off(DK_LED2);
       }
   }
   
   static void right_button_callback(int button_id, bool pressed)
   {
       LOG_INF("Right mouse button %s", pressed ? "pressed" : "released");
       if (pressed) {
           dk_set_led_on(DK_LED3);
       } else {
           dk_set_led_off(DK_LED3);
       }
   }
   
   void main(void)
   {
       int err;
       struct pmw3360_motion_data motion_data;
       
       LOG_INF("Starting nRF52840 Mouse with PMW3360 Sensor");
       
       // Initialize LEDs and buttons
       err = dk_leds_init();
       if (err) {
           LOG_ERR("Cannot init LEDs (err: %d)", err);
           return;
       }
       
       err = dk_buttons_init(NULL);
       if (err) {
           LOG_ERR("Cannot init buttons (err: %d)", err);
           return;
       }
       
       // Initialize custom button handler
       button_handler_init();
       button_handler_register_callback(0, left_button_callback);  // Button 1 = Left click
       button_handler_register_callback(1, right_button_callback); // Button 2 = Right click
       
       // Initialize sensor
       if (!device_is_ready(sensor_dev)) {
           LOG_ERR("PMW3360 sensor device not ready");
           return;
       }
       
       err = pmw3360_init(sensor_dev);
       if (err) {
           LOG_ERR("PMW3360 initialization failed: %d", err);
           return;
       }
       
       // Set CPI to 1600 (good for testing)
       err = pmw3360_set_cpi(sensor_dev, 1600);
       if (err) {
           LOG_ERR("Failed to set CPI: %d", err);
       }
       
       LOG_INF("Mouse initialization complete - starting main loop");
       
       // Main loop
       while (1) {
           // Read sensor data
           err = pmw3360_read_motion(sensor_dev, &motion_data);
           if (err == 0 && motion_data.motion & 0x80) {
               // Motion detected
               LOG_INF("Motion: X=%d, Y=%d, Quality=%d", 
                       motion_data.delta_x, motion_data.delta_y, motion_data.surface_quality);
               
               // Blink LED4 to show motion detection
               dk_set_led_on(DK_LED4);
               k_sleep(K_MSEC(10));
               dk_set_led_off(DK_LED4);
           }
           
           k_sleep(K_MSEC(SENSOR_POLL_INTERVAL_MS));
       }
   }
   ```

2. **Update `CMakeLists.txt`**
   ```cmake
   cmake_minimum_required(VERSION 3.20.0)
   
   find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
   project(mouse_basic)
   
   target_sources(app PRIVATE 
       src/main.c
       src/button_handler.c
       src/pmw3360.c
   )
   
   target_include_directories(app PRIVATE src)
   ```

3. **Build and Test**
   ```bash
   west build -b nrf52840dk_nrf52840 --pristine
   west flash
   ```

4. **Verify Sensor Operation**
   - Open serial terminal
   - Move the sensor over a surface
   - You should see motion log messages
   - LED4 should blink when motion is detected
   - Press buttons to test button functionality

---

## 🎯 **Section 2.3: Basic Bluetooth HID**

### **Step 4: Configure Bluetooth Stack**

#### **4.1 Update Configuration for Bluetooth**

1. **Update `prj.conf`**
   ```ini
   # Previous configs...
   CONFIG_GPIO=y
   CONFIG_LOG=y
   CONFIG_LOG_DEFAULT_LEVEL=3
   CONFIG_CONSOLE=y
   CONFIG_UART_CONSOLE=y
   CONFIG_DK_LIBRARY=y
   CONFIG_SPI=y
   CONFIG_SPI_NRFX=y
   CONFIG_SENSOR=y
   
   # Bluetooth configuration
   CONFIG_BT=y
   CONFIG_BT_PERIPHERAL=y
   CONFIG_BT_DEVICE_NAME="nRF Mouse"
   CONFIG_BT_DEVICE_APPEARANCE=962
   CONFIG_BT_MAX_CONN=1
   CONFIG_BT_MAX_PAIRED=1
   
   # Bluetooth HID configuration
   CONFIG_BT_HIDS=y
   CONFIG_BT_HIDS_DEFAULT_PERM_RW_ENCRYPT=y
   CONFIG_BT_HIDS_ATTR_MAX=10
   
   # Security
   CONFIG_BT_SMP=y
   CONFIG_BT_SIGNING=y
   CONFIG_BT_BONDABLE=y
   CONFIG_BT_MITM=y
   CONFIG_BT_PRIVACY=y
   CONFIG_BT_RPA=y
   
   # Power management
   CONFIG_BT_CONN_CTX=y
   
   # Settings (for bonding)
   CONFIG_SETTINGS=y
   CONFIG_SETTINGS_NVS=y
   CONFIG_NVS=y
   CONFIG_FLASH=y
   CONFIG_FLASH_PAGE_LAYOUT=y
   CONFIG_FLASH_MAP=y
   CONFIG_MPU_ALLOW_FLASH_WRITE=y
   ```

#### **4.2 Create HID Report Descriptor**

1. **Create `src/hid_mouse.h`**
   ```c
   #ifndef HID_MOUSE_H
   #define HID_MOUSE_H
   
   #include <zephyr/types.h>
   
   // HID Report IDs
   #define MOUSE_REPORT_ID     1
   
   // Mouse button masks
   #define MOUSE_BTN_LEFT      BIT(0)
   #define MOUSE_BTN_RIGHT     BIT(1)
   #define MOUSE_BTN_MIDDLE    BIT(2)
   #define MOUSE_BTN_BACK      BIT(3)
   #define MOUSE_BTN_FORWARD   BIT(4)
   
   // Mouse report structure
   struct mouse_report {
       uint8_t buttons;    // Button state bitmask
       int16_t x;          // X movement
       int16_t y;          // Y movement
       int8_t wheel;       // Scroll wheel
   } __packed;
   
   // HID Report Descriptor for mouse
   static const uint8_t hid_mouse_report_desc[] = {
       0x05, 0x01,         // Usage Page (Generic Desktop)
       0x09, 0x02,         // Usage (Mouse)
       0xA1, 0x01,         // Collection (Application)
       0x85, MOUSE_REPORT_ID, // Report ID (1)
       0x09, 0x01,         //   Usage (Pointer)
       0xA1, 0x00,         //   Collection (Physical)
       0x05, 0x09,         //     Usage Page (Buttons)
       0x19, 0x01,         //     Usage Minimum (Button 1)
       0x29, 0x05,         //     Usage Maximum (Button 5)
       0x15, 0x00,         //     Logical Minimum (0)
       0x25, 0x01,         //     Logical Maximum (1)
       0x95, 0x05,         //     Report Count (5)
       0x75, 0x01,         //     Report Size (1)
       0x81, 0x02,         //     Input (Data, Variable, Absolute)
       0x95, 0x01,         //     Report Count (1)
       0x75, 0x03,         //     Report Size (3)
       0x81, 0x01,         //     Input (Constant) - Padding
       0x05, 0x01,         //     Usage Page (Generic Desktop)
       0x09, 0x30,         //     Usage (X)
       0x09, 0x31,         //     Usage (Y)
       0x16, 0x01, 0x80,   //     Logical Minimum (-32767)
       0x26, 0xFF, 0x7F,   //     Logical Maximum (32767)
       0x75, 0x10,         //     Report Size (16)
       0x95, 0x02,         //     Report Count (2)
       0x81, 0x06,         //     Input (Data, Variable, Relative)
       0x09, 0x38,         //     Usage (Wheel)
       0x15, 0x81,         //     Logical Minimum (-127)
       0x25, 0x7F,         //     Logical Maximum (127)
       0x75, 0x08,         //     Report Size (8)
       0x95, 0x01,         //     Report Count (1)
       0x81, 0x06,         //     Input (Data, Variable, Relative)
       0xC0,               //   End Collection (Physical)
       0xC0                // End Collection (Application)
   };
   
   int hid_mouse_init(void);
   int hid_mouse_send_report(const struct mouse_report *report);
   void hid_mouse_connected(void);
   void hid_mouse_disconnected(void);
   
   #endif /* HID_MOUSE_H */
   ```

#### **4.3 Implement Bluetooth HID Service**

1. **Create `src/hid_mouse.c`**
   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/logging/log.h>
   #include <zephyr/bluetooth/bluetooth.h>
   #include <zephyr/bluetooth/hci.h>
   #include <zephyr/bluetooth/conn.h>
   #include <zephyr/bluetooth/uuid.h>
   #include <zephyr/bluetooth/gatt.h>
   #include <bluetooth/services/hids.h>
   #include <zephyr/settings/settings.h>
   #include "hid_mouse.h"
   
   LOG_MODULE_REGISTER(hid_mouse, LOG_LEVEL_DBG);
   
   static struct bt_conn *current_conn = NULL;
   static bool hid_ready = false;
   
   // HID Information
   static const struct hids_info hid_info = {
       .version = 0x0000,
       .code = 0x00,
       .flags = HIDS_REMOTE_WAKE | HIDS_NORMALLY_CONNECTABLE,
   };
   
   // HID Report Map
   static struct hids_report hid_mouse_input_report = {
       .id = MOUSE_REPORT_ID,
       .type = HIDS_INPUT,
   };
   
   // HIDS callbacks
   static void hids_ready_cb(bool ready)
   {
       hid_ready = ready;
       LOG_INF("HID service %s", ready ? "ready" : "not ready");
   }
   
   static void hids_pm_evt_handler(enum hids_pm_evt evt, struct bt_conn *conn)
   {
       switch (evt) {
       case HIDS_PM_EVT_BOOT_MODE_ENTERED:
           LOG_INF("Boot mode entered");
           break;
       case HIDS_PM_EVT_REPORT_MODE_ENTERED:
           LOG_INF("Report mode entered");
           break;
       default:
           break;
       }
   }
   
   // Connection callbacks
   static void connected(struct bt_conn *conn, uint8_t err)
   {
       char addr[BT_ADDR_LE_STR_LEN];
       
       bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
       
       if (err) {
           LOG_ERR("Failed to connect to %s (%u)", addr, err);
           return;
       }
       
       LOG_INF("Connected to %s", addr);
       
       current_conn = bt_conn_ref(conn);
       hid_mouse_connected();
   }
   
   static void disconnected(struct bt_conn *conn, uint8_t reason)
   {
       char addr[BT_ADDR_LE_STR_LEN];
       
       bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
       
       LOG_INF("Disconnected from %s (reason 0x%02x)", addr, reason);
       
       if (current_conn) {
           bt_conn_unref(current_conn);
           current_conn = NULL;
       }
       
       hid_mouse_disconnected();
   }
   
   static void security_changed(struct bt_conn *conn, bt_security_t level, enum bt_security_err err)
   {
       char addr[BT_ADDR_LE_STR_LEN];
       
       bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
       
       if (!err) {
           LOG_INF("Security changed: %s level %u", addr, level);
       } else {
           LOG_WRN("Security failed: %s level %u err %d", addr, level, err);
       }
   }
   
   BT_CONN_CB_DEFINE(conn_callbacks) = {
       .connected = connected,
       .disconnected = disconnected,
       .security_changed = security_changed,
   };
   
   // Advertising data
   static const struct bt_data ad[] = {
       BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
       BT_DATA_BYTES(BT_DATA_UUID16_ALL, BT_UUID_16_ENCODE(BT_UUID_HIDS_VAL)),
       BT_DATA(BT_DATA_NAME_COMPLETE, CONFIG_BT_DEVICE_NAME, sizeof(CONFIG_BT_DEVICE_NAME) - 1),
   };
   
   static const struct bt_data sd[] = {
       BT_DATA_BYTES(BT_DATA_UUID16_ALL, BT_UUID_16_ENCODE(BT_UUID_HIDS_VAL)),
   };
   
   int hid_mouse_init(void)
   {
       int err;
       
       LOG_INF("Initializing Bluetooth HID Mouse");
       
       // Initialize Bluetooth
       err = bt_enable(NULL);
       if (err) {
           LOG_ERR("Bluetooth init failed (err %d)", err);
           return err;
       }
       
       LOG_INF("Bluetooth initialized");
       
       // Load settings
       if (IS_ENABLED(CONFIG_SETTINGS)) {
           settings_load();
       }
       
       // Initialize HIDS
       struct hids_init_param hids_init_param = {
           .info = &hid_info,
           .inp_rep_group_init = {
               .cnt = 1,
               .reports = &hid_mouse_input_report,
           },
           .rep_map = {
               .data = hid_mouse_report_desc,
               .size = sizeof(hid_mouse_report_desc),
           },
           .ready_cb = hids_ready_cb,
           .pm_evt_handler = hids_pm_evt_handler,
       };
       
       err = bt_hids_init(&hids_init_param);
       if (err) {
           LOG_ERR("HIDS init failed (err %d)", err);
           return err;
       }
       
       // Start advertising
       err = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad), sd, ARRAY_SIZE(sd));
       if (err) {
           LOG_ERR("Advertising failed to start (err %d)", err);
           return err;
       }
       
       LOG_INF("Bluetooth HID Mouse initialized - advertising started");
       return 0;
   }
   
   int hid_mouse_send_report(const struct mouse_report *report)
   {
       if (!current_conn || !hid_ready) {
           return -ENOTCONN;
       }
       
       return bt_hids_inp_rep_send(&hids_obj, current_conn, MOUSE_REPORT_ID,
                                   (uint8_t *)report, sizeof(*report), NULL);
   }
   
   void hid_mouse_connected(void)
   {
       LOG_INF("Mouse connected to host");
   }
   
   void hid_mouse_disconnected(void)
   {
       LOG_INF("Mouse disconnected from host");
   }
   ```

### **Step 5: Integrate Everything**

#### **5.1 Create Complete Mouse Application**

1. **Update `src/main.c` with Full Mouse Functionality**
   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/device.h>
   #include <zephyr/drivers/gpio.h>
   #include <zephyr/logging/log.h>
   #include <dk_buttons_and_leds.h>
   #include "pmw3360.h"
   #include "button_handler.h"
   #include "hid_mouse.h"
   
   LOG_MODULE_REGISTER(mouse_main, LOG_LEVEL_INF);
   
   #define SENSOR_POLL_INTERVAL_MS 8  // 125Hz polling rate
   #define MOVEMENT_SCALE_FACTOR   1  // Adjust for sensitivity
   
   // Get sensor device from device tree
   #define PMW3360_NODE DT_NODELABEL(pmw3360)
   static const struct device *sensor_dev = DEVICE_DT_GET(PMW3360_NODE);
   
   // Mouse state
   static struct mouse_report current_report = {0};
   static bool mouse_connected = false;
   
   // Button callbacks
   static void left_button_callback(int button_id, bool pressed)
   {
       LOG_DBG("Left mouse button %s", pressed ? "pressed" : "released");
       
       if (pressed) {
           current_report.buttons |= MOUSE_BTN_LEFT;
           dk_set_led_on(DK_LED2);
       } else {
           current_report.buttons &= ~MOUSE_BTN_LEFT;
           dk_set_led_off(DK_LED2);
       }
       
       if (mouse_connected) {
           hid_mouse_send_report(&current_report);
       }
   }
   
   static void right_button_callback(int button_id, bool pressed)
   {
       LOG_DBG("Right mouse button %s", pressed ? "pressed" : "released");
       
       if (pressed) {
           current_report.buttons |= MOUSE_BTN_RIGHT;
           dk_set_led_on(DK_LED3);
       } else {
           current_report.buttons &= ~MOUSE_BTN_RIGHT;
           dk_set_led_off(DK_LED3);
       }
       
       if (mouse_connected) {
           hid_mouse_send_report(&current_report);
       }
   }
   
   // Override HID connection callbacks
   void hid_mouse_connected(void)
   {
       mouse_connected = true;
       dk_set_led_on(DK_LED1);
       LOG_INF("Mouse connected to host - ready to use");
   }
   
   void hid_mouse_disconnected(void)
   {
       mouse_connected = false;
       dk_set_led_off(DK_LED1);
       LOG_INF("Mouse disconnected from host");
   }
   
   void main(void)
   {
       int err;
       struct pmw3360_motion_data motion_data;
       
       LOG_INF("Starting nRF52840 Bluetooth HID Mouse");
       
       // Initialize LEDs and buttons
       err = dk_leds_init();
       if (err) {
           LOG_ERR("Cannot init LEDs (err: %d)", err);
           return;
       }
       
       err = dk_buttons_init(NULL);
       if (err) {
           LOG_ERR("Cannot init buttons (err: %d)", err);
           return;
       }
       
       // Initialize custom button handler
       button_handler_init();
       button_handler_register_callback(0, left_button_callback);  // Button 1 = Left click
       button_handler_register_callback(1, right_button_callback); // Button 2 = Right click
       
       // Initialize sensor
       if (!device_is_ready(sensor_dev)) {
           LOG_ERR("PMW3360 sensor device not ready");
           return;
       }
       
       err = pmw3360_init(sensor_dev);
       if (err) {
           LOG_ERR("PMW3360 initialization failed: %d", err);
           return;
       }
       
       // Set CPI to 1600 (good default for mouse)
       err = pmw3360_set_cpi(sensor_dev, 1600);
       if (err) {
           LOG_ERR("Failed to set CPI: %d", err);
       }
       
       // Initialize Bluetooth HID
       err = hid_mouse_init();
       if (err) {
           LOG_ERR("HID mouse initialization failed: %d", err);
           return;
       }
       
       LOG_INF("Mouse initialization complete - waiting for connection");
       
       // Main loop
       while (1) {
           // Read sensor data
           err = pmw3360_read_motion(sensor_dev, &motion_data);
           if (err == 0 && (motion_data.motion & 0x80)) {
               // Motion detected
               current_report.x = motion_data.delta_x * MOVEMENT_SCALE_FACTOR;
               current_report.y = motion_data.delta_y * MOVEMENT_SCALE_FACTOR;
               current_report.wheel = 0; // No scroll wheel in this basic version
               
               if (mouse_connected) {
                   err = hid_mouse_send_report(&current_report);
                   if (err == 0) {
                       LOG_DBG("Motion sent: X=%d, Y=%d", current_report.x, current_report.y);
                       dk_set_led_on(DK_LED4);
                       k_sleep(K_MSEC(1));
                       dk_set_led_off(DK_LED4);
                   }
               }
               
               // Reset movement for next report
               current_report.x = 0;
               current_report.y = 0;
           }
           
           k_sleep(K_MSEC(SENSOR_POLL_INTERVAL_MS));
       }
   }
   ```

2. **Update `CMakeLists.txt`**
   ```cmake
   cmake_minimum_required(VERSION 3.20.0)
   
   find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
   project(mouse_basic)
   
   target_sources(app PRIVATE 
       src/main.c
       src/button_handler.c
       src/pmw3360.c
       src/hid_mouse.c
   )
   
   target_include_directories(app PRIVATE src)
   ```

#### **5.2 Build and Test Complete Mouse**

1. **Build the Project**
   ```bash
   west build -b nrf52840dk_nrf52840 --pristine
   west flash
   ```

2. **Test the Mouse**
   - Open serial terminal to monitor logs
   - On your computer, go to Bluetooth settings
   - Look for "nRF Mouse" in available devices
   - Pair with the device
   - Once connected, LED1 should turn on
   - Move the sensor - cursor should move on screen
   - Press Button 1 - should perform left click
   - Press Button 2 - should perform right click

3. **Troubleshooting**
   ```bash
   # If build fails, clean and rebuild
   west build -t clean
   west build -b nrf52840dk_nrf52840
   
   # If flashing fails, erase and reflash
   nrfjprog --eraseall
   west flash
   
   # Check serial output for errors
   # Use nRF Connect for Desktop Serial Terminal
   ```

---

## ✅ **Phase 2 Completion Checklist**

- [ ] **Section 2.1 Complete**
  - [ ] LED blink working
  - [ ] Button input working
  - [ ] Button debouncing implemented
  - [ ] UART logging working

- [ ] **Section 2.2 Complete**
  - [ ] PMW3360 sensor wired correctly
  - [ ] SPI communication working
  - [ ] Motion detection working
  - [ ] CPI setting functional

- [ ] **Section 2.3 Complete**
  - [ ] Bluetooth stack configured
  - [ ] HID service implemented
  - [ ] Device pairing successful
  - [ ] Mouse movement working
  - [ ] Button clicks working

---

## 🎯 **Expected Results**

After completing Phase 2, you should have:

1. **Working Hardware**
   - nRF52840DK with PMW3360 sensor
   - Functional left/right mouse buttons
   - LED indicators for connection and activity

2. **Software Functionality**
   - Bluetooth HID mouse that pairs with computers
   - Accurate motion tracking
   - Responsive button clicks
   - 125Hz polling rate

3. **Development Skills**
   - SPI communication implementation
   - Bluetooth HID protocol understanding
   - Embedded C programming with Zephyr
   - Hardware debugging techniques

---

## 🚀 **Next Steps to Phase 3**

With Phase 2 complete, you're ready to move to Phase 3 where you'll add:
- Advanced sensor features (DPI switching, surface calibration)
- Power optimization
- Professional button handling
- Enhanced performance tuning

**Congratulations on completing Phase 2!** You now have a functional wireless mouse prototype. 🎉
