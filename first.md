# diansai
#include "freertos/FreeRTOS.h"  
#include "freertos/task.h"  
#include "driver/adc.h"  
#include "driver/gpio.h"  
#include "esp_log.h"  
  
#define ADC_CHANNEL_TEMP  ADC1_CHANNEL_0  // LM35连接到ADC1的通道0  
#define ADC_ATTEN_DB_6    ADC_ATTEN_6db    // ADC衰减6dB  
#define FAN_GPIO          18              // 风扇连接到GPIO18  
#define TEMP_THRESHOLD    300             // 温度阈值（根据LM35的校准和ADC的分辨率计算得出）  
  
static const char *TAG = "temp_control";  
  
void read_temperature_and_control_fan(void *pvParameters)  
{  
    adc1_config_width(ADC_WIDTH_BIT_12);  
    adc1_config_channel_atten(ADC_CHANNEL_TEMP, ADC_ATTEN_DB_6);  
  
    gpio_pad_select_gpio(FAN_GPIO);  
    gpio_set_direction(FAN_GPIO, GPIO_MODE_OUTPUT);  
  
    while (1)  
    {  
        uint32_t raw_value = adc1_get_raw(ADC_CHANNEL_TEMP);  
        float voltage = raw_value * (3.3 / 4096.0); // 3.3V是ESP32的ADC参考电压，4096是12位ADC的分辨率  
        float temperature = voltage * 100.0; // LM35的输出电压与温度成正比，每摄氏度10mV  
  
        ESP_LOGI(TAG, "Raw ADC: %u, Voltage: %.2fV, Temperature: %.2fC", raw_value, voltage, temperature);  
  
        if (temperature >= TEMP_THRESHOLD)  
        {  
            gpio_set_level(FAN_GPIO, 1); // 打开风扇  
        }  
        else  
        {  
            gpio_set_level(FAN_GPIO, 0); // 关闭风扇  
        }  
  
        vTaskDelay(pdMS_TO_TICKS(2000)); // 延时2秒  
    }  
}  
  
void app_main(void)  
{  
    xTaskCreatePinnedToCore(  
        read_temperature_and_control_fan,    // 任务函数  
        "temp_control_task",                 // 任务名称  
        configMINIMAL_STACK_SIZE * 2,        // 任务堆栈大小  
        NULL,                                // 任务输入参数  
        5,                                   // 任务优先级  
        NULL,                                // 任务句柄（用于删除任务）  
        0);                                  // 运行在核心0上  
}
