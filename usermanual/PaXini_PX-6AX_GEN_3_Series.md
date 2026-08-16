# PaXini PX-6AX GEN 3 Series Multidimensional Tactile Sensor

User Manual 

## Model

DP -S1813-Elite PXSR-STDDP03F 

DP -S2015-Elite PXSR-STDDP03G 

IP -S1610-Elite PXSR-STDIP03B 

MC -M2020-Elite PXSR-STDMC03A 

DP -S1813-Core PXSR-STDDP03D 

DP -S2716-Core PXSR-STDDP03C 

DP -S3013-Core PXSR-STDDP03E 

IP -M2324-Core PXSR-STDIP03A 

CP -M3025-Core PXSR-STDCP03B 

DP -M2826-Omega PXSR-STDDP03B 

DP -L3530-Omega PXSR-STDDP03A 

CP -L5325-Omega PXSR-STDCP03A 

Catalogue
1. Overview 1
1.1 Introduction 1
1.2 Specification 2
2. Before You Start 5
2.1 Precautions 5
2.2 Security Warning 5
2.3 Disclaimer 6
2.4 Unboxing Inspection 6
3. Dimensions and Installation 7
3.1 Appearance and Dimensions 7
3.2 Coordinate System 9
3.3 Device Connection 10
4. Host Computer Software 11
4.1 Data Visualization 11
4.2 Data Recording 12
4.3 Firmware Update 12
5. Communication Protocol 12
5.1 Pin Definition 12
5.2 Selection Of Communication Protocols 14
5.2.1 Protocol Initialization 14
5.2.2 Settings Protocols 14
5.3 SPI Communication Protocol 14
5.3.1 SPI Film Selection Definition 14
5.3.2 SPI Configuration Requirements 15
5.3.3 SPI Read/ Write Timing 15 

5.3.4 Protocol Format 15  
5.3.5 Instruction 15  
5.4 UART Communication Protocol 16  
5.4.1 UART Setting Requirement 16  
5.4.2 UART Protocol Format 16  
5.4.3 UART Command Example 17  
5.5 IIC Communication Protocol 17  
5.5.1 IIC Setting Requirement 17  
5.5.2 IIC Protocol Format 17  
5.5.3 IIC Command Example 18  
5.6 Address Description 19  
5.6.1 User Configuration Area 19  
5.6.2 Application Area 19  
6. Maintenance and Troubleshooting 20  
6.1 Daily Maintenance and Surface Cleaning 20  
6.2 Troubleshooting 20  
7. Technical Support 20 

## 1. Overview

## 1.1 Introduction

The Paxini PX-6AX GEN3 Multidimensional Tactile Sensors are highly integrated sensors feature threedimensional tactile capabilities, integrating several key functions within a single compact unit. These functions include sensor signal sampling, signal processing, multidimensional tactile algorithms, and communication transmission. 

Main Features: The integration of this sensor into a robotic hand facilitates the collection and interpretation of tactile data, enhancing the robot’s sensory capabilities. This advancement allows robots to discern surface textures and object interactions, leading to improved environmental awareness and refined manipulation skills. The sensor’s versatility supports a wide array of tasks and environments, while its sophisticated hardware and algorithms enable adaptive and intelligent robotic operations. 

Product Name: Paxini PX-6AX GEN3 Multidimensional Tactile Sensor 

Product Model: 

DP-S1813-Elite PXSR-STDDP03F DP-S2015-Elite PXSR-STDDP03G IP -S1610-Elite PXSR-STDIP03B MC-M2020-Elite PXSR-STDMC03A 

DP-S1813-Core PXSR-STDDP03D DP-S2716-Core PXSR-STDDP03C DP-S3013-Core PXSR-STDDP03E IP -M2324-Core PXSR-STDIP03A CP-M3025-Core PXSR-STDCP03B 

DP-M2826-Omega PXSR-STDDP03B DP-L3530-Omega PXSR-STDDP03A CP-L5325-Omega PXSR-STDCP03A 

How It Works: PX-6AX GEN3 Multidimensional Tactile Sensor Series operates on a semi-flexible design that leverages the Hall effect for precise force distribution measurements and multidimensional force data. It features a dual-layered architecture with a pliable layer for deformation-induced magnetic field alterations and a rigid sensing layer for high-fidelity force and tactile data acquisition, processed through an integrated algorithm. The tactile sensor’s Anti-Stray Magnetic Field Function discerns and isolates force-induced magnetic signals from environmental interference. Utilizing advanced algorithms, the Hall sensors’ data is refined to eliminate external magnetic influences, guaranteeing the integrity and precision of the signal outputs. 

## 1.2 Specification

<table><tr><td>Model</td><td>DP-S1813-Elite PXSR-STDDP03F</td><td>DP-S2015-Elite PXSR-STDDP03G</td><td>IP -S1610-Elite PXSR-STDIP03B</td><td>MC-M2020-Elite PXSR-STDMC03A</td></tr><tr><td>Output Signals</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td></tr><tr><td>Dimensions (mm)</td><td>18x13x10</td><td>20x15x10</td><td>16x10x8</td><td>20x20x8</td></tr><tr><td>Number of Force Signals in Array</td><td>31</td><td>52</td><td>25</td><td>9</td></tr><tr><td>Range of Force</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td></tr><tr><td>Measurement Accuracy</td><td>1% FS</td><td>1% FS</td><td>1% FS</td><td>1% FS</td></tr><tr><td>Minimum Detectable Force</td><td>0.1 N</td><td>0.1 N</td><td>0.1 N</td><td>0.1 N</td></tr><tr><td>Spatial Resolution</td><td>1 mm</td><td>1 mm</td><td>1 mm</td><td>1 mm</td></tr><tr><td>Anti-Magnetic</td><td>YES *</td><td>YES *</td><td>YES *</td><td>YES *</td></tr><tr><td>Output Frequency</td><td>83.3 Hz</td><td>83.3 Hz</td><td>83.3 Hz</td><td>83.3 Hz</td></tr><tr><td>Safe Load</td><td>200%</td><td>200%</td><td>200%</td><td>200%</td></tr><tr><td>Shock Overload</td><td>300%</td><td>300%</td><td>300%</td><td>300%</td></tr><tr><td>Service Life</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td></tr><tr><td>Operating Voltage</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td></tr><tr><td>Maximum Operating Current</td><td>150 mA</td><td>150 mA</td><td>150 mA</td><td>100 mA</td></tr></table>


*Please refer to the oficial experimental specifications provided by Paxini. 


<table><tr><td>Model</td><td>DP-S1813-Core PXSR-STDDP03D</td><td>DP-S2716-Core PXSR-STDDP03C</td><td>DP-S3013-Core PXSR-STDDP03E</td><td>IP-M2324-Core PXSR-STDIP03A</td></tr><tr><td>Output Signals</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td></tr><tr><td>Dimensions (mm)</td><td>18x13x10</td><td>27x16x13</td><td>30x13x10</td><td>23x24x8</td></tr><tr><td>Number of Force Signals in Array</td><td>51</td><td>116</td><td>96</td><td>68</td></tr><tr><td>Range of Force</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td></tr><tr><td>Measurement Accuracy</td><td>1% FS</td><td>1% FS</td><td>1% FS</td><td>1% FS</td></tr><tr><td>Minimum Detectable Force</td><td>0.1 N</td><td>0.1 N</td><td>0.1 N</td><td>0.1 N</td></tr><tr><td>Spatial Resolution</td><td>1 mm</td><td>1 mm</td><td>1 mm</td><td>1 mm</td></tr><tr><td>Anti-Magnetic</td><td>YES *</td><td>YES *</td><td>YES *</td><td>YES *</td></tr><tr><td>Output Frequency</td><td>83.3 Hz</td><td>83.3 Hz</td><td>83.3 Hz</td><td>83.3 Hz</td></tr><tr><td>Safe Load</td><td>200%</td><td>200%</td><td>200%</td><td>200%</td></tr><tr><td>Shock Overload</td><td>300%</td><td>300%</td><td>300%</td><td>300%</td></tr><tr><td>Service Life</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td></tr><tr><td>Operating Voltage</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td></tr><tr><td>Maximum Operating Current</td><td>200 mA</td><td>300 mA</td><td>300 mA</td><td>350 mA</td></tr><tr><td>Model</td><td>CP-M3025-Core PXSR-STDCP03B</td><td>DP-M2826-Omega PXSR-STDDP03B</td><td>DP-L3530-Omega PXSR-STDDP03A</td><td>CP-L5325-Omega PXSR-STDCP03A</td></tr><tr><td>Output Signals</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td><td>Triaxial force array and Triaxial resultant force</td></tr><tr><td>Dimensions (mm)</td><td>30x25x8</td><td>28x26x14</td><td>35x30x15</td><td>53x25x8</td></tr><tr><td>Number of Force Signals in Array</td><td>77</td><td>127</td><td>135</td><td>239</td></tr><tr><td>Range of Force</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td><td>Normal Fz:0 ~ 25 N Tangential Fxy:± 10 N</td></tr><tr><td>Measurement Accuracy</td><td>1% FS</td><td>1% FS</td><td>1% FS</td><td>1% FS</td></tr><tr><td>Minimum Detectable Force</td><td>0.1 N</td><td>0.1 N</td><td>0.1 N</td><td>0.1 N</td></tr><tr><td>Spatial Resolution</td><td>1 mm</td><td>1 mm</td><td>1 mm</td><td>1 mm</td></tr><tr><td>Anti-Magnetic</td><td>YES *</td><td>YES *</td><td>YES *</td><td>YES *</td></tr><tr><td>Output Frequency</td><td>83.3 Hz</td><td>83.3 Hz</td><td>83.3 Hz</td><td>83.3 Hz</td></tr><tr><td>Safe Load</td><td>200%</td><td>200%</td><td>200%</td><td>200%</td></tr><tr><td>Shock Overload</td><td>300%</td><td>300%</td><td>300%</td><td>300%</td></tr><tr><td>Service Life</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td><td>Supports over 10 million cycles*</td></tr><tr><td>Operating Voltage</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td><td>3 ~ 5 V</td></tr><tr><td>Maximum Operating Current</td><td>350 mA</td><td>400 mA</td><td>400 mA</td><td>700 mA</td></tr></table>

## 2 . Before You Start

## 2.1 Precautions

When using tactile sensors, here are some considerations to keep in mind: 

1. Cleaning and Maintenance: Regularly clean the tactile sensors to remove dust, grease, or stains. Follow the cleaning guidelines provided by the manufacturer, using appropriate cleaning agents and tools. Avoid using overly abrasive cleaners to prevent damage to the sensor. 

2. Environmental Adaptation: Tactile sensors may be sensitive to environmental conditions. Ensure that the sensors are used under suitable temperature, humidity, and other environmental conditions. Avoid exposure to excessively high or low temperatures, humidity levels, or other adverse environmental factors. 

3. Waterproof Protection: If the tactile sensor is used in damp or liquid environments, ensure that the sensor has appropriate waterproof protection. Check that the seals and connections of the sensor are intact, and take necessary protective measures, such as using waterproof covers or sealing compounds. In addition to waterproof protection for the sensor itself, the connector should also be sealed and protected. 

4. Pay Attention to Power and Electrical Connections: When connecting the power and electrica circuits of the tactile sensor, ensure proper connections and follow relevant safety standards and regulations. Avoid power overload, short circuits, or other electrical issues. 

5. Follow Manufacturer Guidelines: Adhere to the operating manual, safety guidelines, and maintenance recommendations provided by the manufacturer. Different models of tactile sensors may have different requirements and characteristics, so always refer to the relevant manufacturer guidelines to ensure correct use and maintenance of the sensor. 

6. Storage and Transportation: If storing or transporting tactile sensors, take care to avoid severe vibrations, compression, or other situations that could cause damage to the sensor. 

Before using the tactile sensor, be sure to thoroughly read the relevant product documentation and知guidelines, and follow the manufacturer's recommendations and instructions.

## 2.2 Security Warning

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/f977e02a5727bb6e2332e68b80b41169d6ca319c2ea5299fe63fc50d1032f6f3.jpg)


Caution: Avoid accidental ingestion by children. 

Caution: Prevent the product from getting soaked in water. 

Caution: Keep the product away from sources of fire. 

Caution: Prevent product debris from splashing into the eyes. 

## 2.3 Disclaimer

This product is a tactile sensor that can be used in normal environments, such as touching, gripping, squeezing, etc. It is not suitable for extreme scenarios, such as poking, burning, knocking, etc. Such actions may damage the functionality and performance of the product, and could even pose a danger. If customers encounter these extreme scenarios while using the product, they should immediately stop using it and contact our customer service personnel. We will do our best to provide you with solutions and services. 

In addition, this product is also not suitable for the following extreme scenarios: 

1. Immersing the tactile sensor in liquids such as water, oil, acids, or alkalis, which may cause short circuits, corrosion, or oxidation. 

2. Exposing the tactile sensor to environments with high temperatures, low temperatures, high humidity, strong light, or strong magnetic fields, which may result in material deformation, aging, or failure. 

3. Connecting the tactile sensor to incompatible power supplies, signal sources, or devices, which may cause excessive voltage, current, or interference. 

4. Cutting, scratching, tearing, or folding the tactile sensor, which may lead to structural damage, electrode breakage, or signal loss. 

5. Using the tactile sensor in unsuitable application scenarios, such as medical, military, or aviation fields, which may lead to safety risks or legal liabilities. 

Our company is not responsible for any loss or injury caused by the customer's violation of this disclaimer. Customers are advised to carefully read and agree to this disclaimer before purchasing and using the product. 

Our company reserves the final right to interpret the product specifications. If you have any questions or objections, please contact us in a timely manner. Thank you for your support and trust. 

## 2.4 Unboxing Inspection

When you receive a new product and proceed with the unboxing inspection, here are some recommended steps: 

1. Check the Packaging:Inspect the product's external packaging to ensure it is intact and undamaged. If the packaging shows obvious signs of damage or deformation, it may indicate that the product was damaged during transit. Before signing of, you can take photos of the packaging condition as evidence for future reference, should you need to present it to the supplier or logistics company. 

2. Verify Accessories: Cross-reference the accessories listed on the product packaging or manual to ensure all components are included in the package, with no omissions or damages. If any accessories are missing or damaged, contact the supplier promptly to address the issue. 

3. Inspect the Product Appearance: Carefully examine the product's appearance to ensure there are no visible scratches, dents, deformations, or other damages. Should any issues arise, immediately contact the supplier and document the damages with photos as evidence. 

4. Safety Precautions: When conducting any tests or operations, always pay attention to safety precautions. Follow the warnings and safety guidelines outlined in the product manual to ensure a safe and reliable operation process. 

If you encounter any problems, damages, or anomalies during the unboxing inspection, it's advisable to immediately contact the supplier or seller, report the issue, and seek a resolution. Keep all related photos, documents, and communication records for future reference if needed. 

## 3.Dimensions and Installation

## 3.1 Appearance and Dimensions

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/155d8a284d4f9e09601e681d07a6ea8928fb3ff5530815998e18acdab2c44db3.jpg)



DP-S1813-Elite PXSR-STDDP03F


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/b36b59c2709ec953cd0d70e6d5976cb68f3295e03386d648202437ce2596d828.jpg)



DP-S2015-Elite PXSR-STDDP03G


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/e07a2160c4519bb4b43e2cae0ed58cb817f1737cd5ea39f0eb1f0f890d9b5578.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/6371c1e541fef38e0742b030f1929158a77d440e55bfabb4fc977774feed6c1b.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/942302001971a55efd2a7cadf84c9d37159bf15b9ce2fb9bec759ff79294dc02.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/2fa19088f2612da2fe29ec9ec0b17551cd3555ef27dd58efcfc304fa9172ebf7.jpg)



IP -S1610-Elite PXSR-STDIP03B


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/f90df50b6017a464792b81fcf79ea1549c7bc2bd10d50093e8e0573cb0bb7b7e.jpg)



DP-S1813-Core PXSR-STDDP03D



MC-M2020-Elite PXSR-STDMC03A


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/732b836ba30d66c3ce3b19b48ebf66d6af8db7ba116e6cbd209c20b29fb14176.jpg)



DP-S2716-Core PXSR-STDDP03C


*The linear dimensional tolerance of the positioning holes is ± 0.05mm, and the tolerance of other unmarked dimensions is ± 0.1mm 

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/4bcc485634396e272344a413a93e17d1b5ae80789b2a6fbcb8048c245759dd2d.jpg)



DP-S3013-Core PXSR-STDDP03E


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/3255bcb38c92aed7f506ce84968f9874e073e64f762157377f9b8ba3cf68991c.jpg)



IP-M2324-Core PXSR-STDIP03A


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/cf5e5ccd67879a7ce2d8c838b3b35bb912d66e39b061a6b319ca071356a48f71.jpg)



CP-M3025-Core PXSR-STDCP03B


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/1b8dd1fc217f9f874344944fd40cfa21d9054f9281ce2aa0322bc974b8290fe5.jpg)



DP-M2826-Omega PXSR-STDDP03B


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/ec27d12dd9a9d3b6776380ed39f35a067b6af66ef210eebe016a3b7393ad7c55.jpg)


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/fcf5c29b3d5cd5dd9693c0d7feb625f5cb7fab6a6bf6972a5118ab080b5d335a.jpg)



DP-L3530-Omega PXSR-STDDP03A


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/8bf957394b2582ff275031ede53c9c00010ff538331fe68af0c983d02aad2a70.jpg)


*The linear dimensional tolerance of the positioning holes is ± 0.05mm, and the tolerance of other unmarked dimensions is ± 0.1mm 

## 3.2 Coordinate System

## Coordinates of Force Signals in Array

The green dot at the tail of the sensor bracket is the origin of the global coordinate system (0,0,0), which is for determining the spatial position of each force signal within the array. The global coordinate position and force signals in array are shown as follows. 

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/badd80e374cec1d7312fafc4b0f6e528403ecfc3813c775d6961ae760d974f93.jpg)



DP-S1813-Elite PXSR-STDDP03F


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/e68dd3e9d3db5c1a0abb8abf53479e12eb25573e6416071506df9a87d0308c50.jpg)



DP-S2015-Elite PXSR-STDDP03G


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/615c170d0a87bb5109cea91952b869402e7fbf51c76ec684faeed520e15b9004.jpg)



IP -S1610-Elite PXSR-STDIP03B


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/f961be36994fc968067db7f724748c4b65cde0e3d3d4b360f5c7f6d292f5dc06.jpg)



MC-M2020-Elite PXSR-STDMC03A


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/3ab56381a0d9acf141cdae40844093e46278eae4b62ec868da24c130fb38e10e.jpg)



DP-S1813-Core PXSR-STDDP03D


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/ea311c25962fb7465c52b2662ad0dbfcde8df8bded419e68ff917e1f230664b5.jpg)



DP-S2716-Core PXSR-STDDP03C


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/0d3dfaf6cd3cbe9ef9ec7974e733e8161ffa5f79acd15f339a9068500fcbf1b0.jpg)



DP-S3013-Core PXSR-STDDP03E


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/f6c09e01e263cdfcd8140f57b37bc9e60fcc43e8b3dadda1e5bfef79605bf091.jpg)



IP-M2324-Core PXSR-STDIP03A


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/940cc34da2e44eb0eea375738cc60d9c2c04b3d29d288031700a65effbd1ff30.jpg)



CP-M3025-Core PXSR-STDCP03B


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/144a135ac67c66dbf3814d2f04b1da772931a1fe9ba5906426ba121b2d68a517.jpg)



DP-M2826-Omega PXSR-STDDP03B


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/43fe38c2d8bf068598177725eee19e4e545d1780d20e785cb3ca932c0a61e67c.jpg)



DP-L3530-Omega PXSR-STDDP03A


![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/bee09a24dcb9a5594477b68167114c8bdf6b03f511f582b68ae1a8acd8b228ab.jpg)



CP-L5325-Omega PXSR-STDCP03A


## 3.3 Device Connection

When using the host computer software, make sure that the sensor is connected to the corresponding communication board. The communication board (optional) is connected to the PC via a USB cable. To update the sensor firmware, there are three methods: 

1. Use the 10-channel SPI hub for flashing; note that the sensor needs to be connected to the CN1 port. 

2. Use the serial converter board for flashing. 

3. Use the high-speed communication board for flashing. 

## 4.Host Computer Software

## 4.1 Data Visualization

After ensuring that the steps in the 3.3 device connection have been completed, start the pxsr-gen3 software. Single sensor connection: 

1. Select the communication board (SPI: 10-channel SPI hub; USB: serial converter board; HAND: high-speed communication board). 

2. Select the COM port. 

3. Click [Open], and the host computer will automatically identify the sensor model and start the connection with the sensor. 

Note: To connect an MC-M2020-Elite sensor via the 10-channel SPI hub, first select the sensor model in pxsrgen3 software prior to clicking [Open]; if connecting to other models of sensors, just confirm that the currently selected model is not MC-M2020. 

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/54d176c9ea39533003acff4fed7871ec24bdb7c802d824895401702a6b4bbc89.jpg)


Please select the serial 

On the right side of the software is the plot of the resultant forces Fx, Fy, Fz over time. 

Note: After the sensor is successfully connected, the version information of the sensor will be displayed in the upper right corner and the resultant force curve will also be shown on the right side. 

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/3b7a971be209a974410531f2628eecc5a53e7a2608a125ee10f75b783f89aeef.jpg)


Make sure that when the sensor has no-load, click [Calibration] and set the sensor to zero for calibration. 

When pressed, the magnitude and the direction尼of the resultant force will be displayed at thecorresponding positions in the UI. The greaterthe force, the larger the points in the dot matrix.

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/644266c71763ff4b0cfe7bbd73b95e1a75c9c562bd3037ab287fedb5635d1238.jpg)


## 4.2 Data Recording

Click the [Start] to begin recording the force data from the sensor. Click [Stop] to cease recording. Clicking the [OpenFile] will open the folder containing the records. 

Start 

① Stop 

OpenFile 

## 4.3 Firmware Update

Click on [Upgrade] to enter the firmware upgrade window. Please reconfirm that the sensor is connected to the CN1 port of the 10-channel SPI hub, or to the serial converter board/high-speed communication board. 

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/c065be033eb6d0840f95a46931f31436538cddf4c6e06e081038a052a1ecfd0c.jpg)


Select [Online update] , and then click [Start Upgrade] to initiate the firmware flashing program. 

Upgrade 

Online update Local update 

The latest firmware version: V1.0.1 

StartUpgrade 

## 5. Communication Protocol

## 5.1 Pin Definition

## DP-S1813-Elite PXSR-STDDP03F

## DP-S2015-Elite

## PXSR-STDDP03G

## DP-S1813-Core知

## PXSR-STDDP03D

## DP-S2716-Core

## PXSR-STDDP03C

DP-S3013-Core PXSR-STDDP03E 

## DP-M2826-OmegaPXSR-STDDP03B

DP-L3530-Omega PXSR-STDDP03A 

$$
3 P = C S 1
$$

$$
4 P = M I S O
$$

Note: VDD is 3 to 5V, 

and the high level of 

$$
5 P = V D D
$$

the other pins is 3.3V 

$$
6 P = M O S I
$$

$$
7 P = G N D
$$

$$
8 P = S C L K
$$

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/3140aaf452b53dcf0f7a695ced1056bf7090345f6cf3a20f2caf76c25d98bd90.jpg)


IP -S1610-Elite PXSR-STDIP03B IP -M2324-Core PXSR-STDIP03A CP-M3025-Core PXSR-STDCP03B CP-L5325-Omega PXSR-STDCP03A 

Note: VDD is 3 to 5V, and the high level of the other pins is 3.3V 

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/bc6740c4e3cc650f6b2cca70836fc1b12637be32da24d8e05fa279651b5287a6.jpg)


## MC-M2020-Elite PXSR-STDMC03A

Connector spec: BM28B0.6-10DS 1P=CS1 2P=CS2 3P=CS3 4P=CS4 5P=CS5 6P=MISO 7P=GDN 8P=MOSI 9P=GND 10P=SCLK 11P=GND 12P=GND 13P=3V3 14P=3V3 

![image](https://cdn-mineru.openxlab.org.cn/result/2026-08-16/68ce4598-0787-489e-a012-f20339bc8d6e/71031b3a017726e432b99bb2351c035a00839bc74c9c3ab01c2cdfa6d8faf531.jpg)



Terminal TOP layer view


## 5.2 Selection Of Communication Protocols

## 5.2.1 Protocol Initialization

Protocol selection and initialization will only be carried out when the module is powered on. Once the power-on is successful and the initialization is completed, the protocol will not change. 

## 5.2.2 Settings Protocols

<table><tr><td>Corresponding protocols</td><td colspan="3">Pin definition</td><td>CS3</td><td>CS2</td><td>CS1</td></tr><tr><td>SPI</td><td>SPI_CLK</td><td>SPI_MISO</td><td>SPI_MOSI</td><td>1</td><td>1</td><td>1</td></tr><tr><td>UART</td><td>UART_TX(slave)</td><td>UART_RX(slave)</td><td></td><td>0</td><td>0</td><td>0</td></tr><tr><td>IIC</td><td></td><td></td><td>SCL</td><td>0</td><td>0</td><td>SDA (1)</td></tr></table>

The order of address encoding values is CS3\CS2\CS1. For example, in IIC connection mode, the levels of CS3\CS2\CS1 are 001 when powered on. 

The sensor automatically selects the corresponding protocol according to the connection status. Note: 

SPI mode: CS3\CS2\CS1 need to be connected to the host GPIO, and CS3\CS2\CS1 need to be pulled up when powered on. 

UART mode: No need to connect CS3\CS2\CS1 or directly pull them down. 

IIC mode: Need to connect CS1 and pull it up, no need to connect CS3\CS2 or directly pull them dowm. 

## 5.3 SPI Communication Protocol

## 5.3.1 SPI Film Selection Definition

CS3\CS2\CS1 are the address decoding pins, the available chip selection number range is 001 to 110. The following are the factory default chip selection numbers. 

<table><tr><td></td><td>CS3</td><td>CS2</td><td>CS1</td></tr><tr><td>DP (fingertip)</td><td>0</td><td>1</td><td>1</td></tr><tr><td>IP (middle finger pad)</td><td>0</td><td>1</td><td>0</td></tr><tr><td>CP (closed finger pad)</td><td>0</td><td>0</td><td>1</td></tr></table>

Palm sensor MC-M2020, CS5\CS4\CS3\CS2\CS1 are address decoding pins, with the default being感00001. Currently, the available chip selection number range is 00001 to 01000.

*In the accompanying software, click on [Settings] to change the module number of the sensor. The module number = chip selection number - 1. 

· Module number setting 

## 5.3.2 SPI Configuration Requirements

SPI_InitStructure.SPI_CPOL = SPI_CPOL_High; //Idle state for serial clock is high level SPI_InitStructure.SPI_CPHA = SPI_CPHA_2Edge; //Data is sampled on the second rising edge of the serial synchronous clock 

Data transmission order: Most Significant Bit First (i.e., bit7 of each byte is transmitted first) It is recommended to connect a 51Ω resistor in series with the clock signal at the host end to enhance the stability of signal transmission 

## 5.3.3 SPI Read/ Write Timing

Write： 

<table><tr><td>Pull Down CS</td><td>Delay</td><td>Function Code + Data, Etc</td><td>Delay</td><td>Raise All CS</td><td>Delay</td></tr><tr><td></td><td>&gt;=8us</td><td></td><td>&gt;=25us</td><td></td><td>&gt;=8us</td></tr></table>

Read： 

<table><tr><td>Pull Down CS</td><td>Delay</td><td>Function Code + Address+ Length</td><td>Delay</td><td>Data,Etc</td><td>Delay</td><td>Raise All CS</td><td>Delay</td></tr><tr><td></td><td>&gt;=8us</td><td></td><td>&gt;=25us</td><td></td><td>&gt;=25us</td><td></td><td>&gt;=8us</td></tr></table>

## 5.3.4 Protocol Format

SPI Read： 

<table><tr><td>0x80 + Function Code</td><td>Start Address</td><td>Read Data Byte Length</td><td>Staus</td><td>Data</td><td>CRC8 Check</td></tr><tr><td>0xFB</td><td>2 bytes, Little-Endian</td><td>2 bytes, Little-Endian</td><td>1 byte</td><td></td><td>1 byte</td></tr></table>

CRC8 Check：Perform CRC check on all preceding bytes 0xFB：The function code 0x7B is obtained after the highest position of 1 

SPI Write： 

<table><tr><td>0x00 +Function Code</td><td>Start Address</td><td>Write Data Byte Length</td><td>Data</td><td>CRC8 Check</td></tr><tr><td>0x79</td><td>2 bytes, Little-Endian</td><td>2 bytes, Little-Endian</td><td></td><td>1 byte</td></tr></table>

CRC8 Check：Perform CRC check on all preceding bytes 

## 5.3.5 Instruction

Write the meanings of the module's function codes, data, and so on： 

<table><tr><td>Position</td><td>Meaning</td><td>Instruction</td></tr><tr><td>bytes 1</td><td>Function Code</td><td>0x79 (User Configuration Area)</td></tr><tr><td>bytes 2~3</td><td>Start Address</td><td>Little-Endian</td></tr><tr><td>bytes 4~5</td><td>Write Data Byte Length</td><td>Little-Endian</td></tr><tr><td>bytes 6</td><td>Data</td><td>Refer to Section 5.6</td></tr><tr><td>bytes 7</td><td>CRC8 Check</td><td>Perform CRC check on all preceding bytes</td></tr></table>


The meaning of reading the module's function code + address + length：


<table><tr><td>Position</td><td>Meaning</td><td>Instruction</td></tr><tr><td>bytes 1</td><td>Function Code</td><td>0x7B (Application Area)</td></tr><tr><td>bytes 2~3</td><td>Start Address</td><td>Little-Endian</td></tr><tr><td>bytes 4~5</td><td>Read Data Byte Length</td><td>Little-Endian, Refer to Section 5.6, For example, when reading the data of three measurement points (Fx, Fy, Fz), the data length = 3*3</td></tr></table>


The meaning of reading data from the module：


<table><tr><td>Position</td><td>Meaning</td><td>Instruction</td></tr><tr><td>bytes 6</td><td>Status</td><td>For debugging, return from the slave machine, no need to pay attention</td></tr><tr><td>bytes 7~...</td><td>Data</td><td>Return from the slave machine</td></tr><tr><td>The last 1 byte</td><td>CRC8 Check</td><td>Perform CRC check on all preceding bytes, Return from the slave machine</td></tr></table>

## 5.4 UART Communication Protocol

## 5.4.1 UART Setting Requirement

Baud rate：921600 

Data bits：8 bit 

Stop bit：1 bit 

Parity check：NONE 

Flow Control：NONE 

It is recommended that users set the host Tx to push-pull mode to enhance the driving capability; set the host Rx to open-drain mode and use 1k-1.5k resistors to pull up the bus. 

This protocol can only be executed through the request -> return mode. 

Device address = module number +1, refer to Section 5.3 

## 5.4.2 UART Protocol Format


Read - Request frame


<table><tr><td colspan="2">Frame Header</td><td>Frame Length</td><td>Device Address</td><td>Reserved</td><td>0x80+Function Code</td><td>Start Address</td><td>Data Length</td><td>LRC Check</td></tr><tr><td colspan="2">data[0-1]</td><td>data[2-3]</td><td>data[4]</td><td>data[5]</td><td>data[6]</td><td>data[7-10]</td><td>data[11-12]</td><td>data[13]</td></tr><tr><td>0x55</td><td>0xAA</td><td>2 bytes, Little</td><td>1 byte</td><td>0x00</td><td>0xFB</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>1 byte</td></tr></table>


Read - Response frame


<table><tr><td colspan="2">Frame Header</td><td>Frame Length</td><td>Device Address</td><td>Reserved</td><td>0x80 +Function Code</td><td>Start Address</td><td>Returned Data Length</td><td>Status</td><td>Returned Data</td><td>LRC Check</td></tr><tr><td colspan="2">data[0-1]</td><td>data[2-3]</td><td>data[4]</td><td>data[5]</td><td>data[6]</td><td>data[7-10]</td><td>data[11-12]</td><td>data[13]</td><td></td><td>data[N+14]</td></tr><tr><td>0xAA</td><td>0x55</td><td>2 bytes, Little</td><td>1 byte</td><td>0x00</td><td>0xFB</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>1 byte</td><td>N bytes</td><td>1 byte</td></tr></table>


LRC Check：Perform LRC check on all preceding bytes 



Write - Request frame


<table><tr><td colspan="2">Frame Header</td><td>Frame Length</td><td>Device Address</td><td>Reserved</td><td>0x00+Function Code</td><td>Address</td><td>Data Length</td><td>Write Data</td><td>LRC Check</td></tr><tr><td colspan="2">data[0-1]</td><td>data[2-3]</td><td>data[4]</td><td>data[5]</td><td>data[6]</td><td>data[7-10]</td><td>data[11-12]</td><td></td><td>data[N+14]</td></tr><tr><td>0x55</td><td>0xAA</td><td>2 bytes, Little</td><td>1 byte</td><td>0x00</td><td>0x79</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>N bytes</td><td>1 byte</td></tr></table>


Write - Response frame


<table><tr><td colspan="2">Frame Header</td><td>Frame Length</td><td>Device Address</td><td>Reserved</td><td>0x00 +Function Code</td><td>The Address For Writing Instructions</td><td>Returned Data Length</td><td>Status</td><td>LRC Check</td></tr><tr><td colspan="2">data[0-1]</td><td>data[2-3]</td><td>data[4]</td><td>data[5]</td><td>data[6]</td><td>data[7-10]</td><td>data[11-12]</td><td>data[13]</td><td>data[14]</td></tr><tr><td>0xAA</td><td>0x55</td><td>2 bytes, Little</td><td>1 byte</td><td>0x00</td><td>0x79</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>1 byte</td><td>1 byte</td></tr></table>


Status： 


0x00: write success 

LRC Check：Perform LRC check on all preceding bytes 

Frame Length: From data[4] to data[N+13] 

W = 0x00 ； R = 0x01 

0xFB：Set the highest bit of function code 0x7B to 1 

## 5.4.3 UART Command Example

Read the first 32 bytes at address 1038 (force signals in array) in the 0x7B area. 

55 AA 09 00 01 00 FB 0E 04 00 00 20 00 CA The equipment address is 01 55 AA 09 00 02 00 FB 0E 04 00 00 20 00 C9 The equipment address is 02 55 AA 09 00 03 00 FB 0E 04 00 00 20 00 C8 The equipment address is 03 

## 5.5 IIC Communication Protocol

## 5.5.1 IIC Setting Requirement

The IIC rate is tentatively set at 200K and below. 

The device address = module number +1. The module number is referred to Section 5.3. 

All data are presented in little-endian mode. 

IIC SDA and SCL need to be set to open-drain mode. In terms of hardware, strong pull-up is required. It is recommended to use a resistor of 4.7K or less for pull-up. 

## 5.5.2 IIC Protocol Format

Read - Request frame 

<table><tr><td>(Device Address &lt;&lt; 1) | W</td><td>0x80 + Function Code</td><td>Start Address</td><td>Data Length</td><td>LRC Check</td></tr><tr><td>data[0]</td><td>data[1]</td><td>data[2-5]</td><td>data[6-7]</td><td>data[8]</td></tr><tr><td>1 byte</td><td>0xFB</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>1 byte</td></tr></table>


Read - Response frame


<table><tr><td>(Device Address &lt;&lt; 1) | R</td><td>0x80 + Function Code</td><td>Start Address</td><td>Returned Data Length</td><td>Status</td><td>Returned Data</td><td>LRC Check</td></tr><tr><td>data[master]</td><td>data[0]</td><td>data[1-4]</td><td>data[5-6]</td><td>data[7]</td><td></td><td>data[N+8]</td></tr><tr><td>1 byte</td><td>0xFB</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>1 byte</td><td>N bytes</td><td>1 byte</td></tr></table>

Data reading process: Request frame sent + waiting delay ≥ 25us + response frame reads data back - Both the request frame and the response frame require the participation of the host, among which: 

- The red part indicates that the host controls the SDA line to send data and the slave receives data 

- The blue part is for the slave control SDA to send data and the host to receive data LRC check: Perform LRC check on all the preceding bytes 


Write - Request frame


<table><tr><td>(Device Address &lt;&lt; 1) | W</td><td>0x00 + Function Code</td><td>Start Address</td><td>Return The Byte Length</td><td>write data</td><td>LRC Check</td></tr><tr><td>data[0]</td><td>data[1]</td><td>data[2-5]</td><td>data[6-7]</td><td></td><td>data[N+8]</td></tr><tr><td>1 byte</td><td>0x79</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>N bytes</td><td>1 byte</td></tr></table>


Write - Response frame


<table><tr><td>(Device Address &lt;&lt; 1) | R</td><td>0x00 + Function Code</td><td>Start Address</td><td>Return the byte length</td><td>Status</td><td>LRC Check</td></tr><tr><td>data[master]</td><td>data[0]</td><td>data[1-4]</td><td>data[5-6]</td><td>data[7]</td><td>data[N+8]</td></tr><tr><td>1 byte</td><td>0x79</td><td>4 bytes, Little</td><td>2 bytes, Little</td><td>1 byte</td><td>1 byte</td></tr></table>

Data reading process: Request frame sent + waiting delay ≥ 25us + response frame reads data back - Both the request frame and the response frame require the participation of the host, among which: 

- The red part indicates that the host controls the SDA line to send data and the slave receives科data

- The blue part is for the slave control SDA to send data and the host to receive data Status： 

0x00: write success 

LRC check: Perform LRC check on all the preceding bytes 

W = 0x00，R = 0x01 

0xFB：Set the highest bit of function code 0x7B to 1 

## 5.5.3 IIC Command Example

The fingertip module number is 02: Read the first 32 bytes at address 1038 (force signals in array) in尼the 0x7B area.

(Request frame) sent by the host: 06 FB 0E 04 00 00 20 00 CD ((Device Address 3) << 1) | 0(Response frame) sent by the host: 07西 ((Device Address 3) << 1) | 1

## 5.6 Address Description

## 5.6.1 User Configuration Area: (Function Code: 0x79) (not continuous write operations)

<table><tr><td>Address</td><td>Register Declaration</td><td>Streamability</td><td>Remarks</td></tr><tr><td>1</td><td>Reserve</td><td></td><td></td></tr><tr><td>2</td><td>Reserve</td><td></td><td></td></tr><tr><td>3</td><td>Calibration. Setting it to 1 is effective. Set it to 0 for normal operation</td><td>W</td><td>Only one byte can be written</td></tr></table>

## 5.6.2 Application Area: (Function Code: 0x7B) (Read-only area, supports continuous reading)

<table><tr><td>1008</td><td>Combined Force Value FX</td><td rowspan="3">Data Range for Fx (-128 to +127), Fy (-128 to 127), Fz (0 to 255)One LSB represents one unit of data resolution.When the resultant force value Fz is read as 10, the magnitude of the force measured by Fz is <eq>10 \times 0.1N = 1.0N</eq></td></tr><tr><td>1009</td><td>Combined Force Value FY</td></tr><tr><td>1010</td><td>Combined Force Value FZ</td></tr><tr><td>1038</td><td>Force on the 1st force in array FX</td><td rowspan="10">A single byte represents the data range Fx (-128 to +127), Fy (-128 to 127), and Fz (0 to 255).</td></tr><tr><td>1039</td><td>Force on the 1st force in array FY</td></tr><tr><td>1040</td><td>Force on the 1st force in array FZ</td></tr><tr><td>1041</td><td>Force on the 2nd force in array FX</td></tr><tr><td>1042</td><td>Force on the 2nd force in array FY</td></tr><tr><td>1...</td><td>...</td></tr><tr><td>1...</td><td>...</td></tr><tr><td>1383*</td><td>Force on the 116th force in array FX</td></tr><tr><td>1384*</td><td>Force on the 116th force in array FY</td></tr><tr><td>1385* ...</td><td>Force on the 116th force in array FZ ...</td></tr></table>

## 6. Maintenance and Troubleshooting

## 6.1 Daily Maintenance and Surface Cleaning

When the sensor's elastomeric surface becomes dusty, it is recommended to gently remove the dust using clear tape. Alternatively, paper towels, lens cleaning cloths, water-based wet wipes, or cotton cloths with a low concentration of alcohol can be used for cleaning. 

The sensor's casing can be cleaned using paper towels, lens cleaning cloths, or water-based wet wipes, but it should not come into contact with alcohol or other organic/non-polar solvents. 

## 6.2 Troubleshooting

## ● The Host Computer Software displays jumbled sensor values

You can verify the actual sensors being accessed by gently pressing the elastomer on each tactile sensor connected to the computer, while observing if the images displayed in the software change with the pressure applied. Check for sources of interference around the tactile sensors, such as pressure points, magnetic fields, etc. If the readout values tend to stabilize without fluctuation and the sensors are not afected by other sources of interference, click the CALIBRATION button to zero the sensors. 

## ● After turning on the sensor, the Host Computer Software is unable to receive data

Please check the following elements in sequence: 

1. Ensure the sensor's Dupont wires are securely connected to the data processing box. 

2. Verify the data processing box is properly connected to the computer, and the LED on the box is green. 

3. Make sure the correct COM port is selected and there is no interference from other COM ports. 

4. The port switch should be set to OPEN PORT. Try toggling the OPEN PORT switch to see if it becomes efective. 

## ● The resultant force calculated from the sum of distributed forces is inaccurate

There are several potential causes for this issue. Please troubleshoot according to the following reasons: 

1. The object applying force contacted areas outside the sensing surface range of the elastomer. 

2. The tactile sensor's casing (except for the mounting connector locations) is subjected to force during operation. 

3. The elastomer is contacted by external objects at a high speed or deeply pressed by sharp objects. 

4. The measuring surface is subjected to a large area of uniformly distributed force in the direction知along the surface.

5. The localized force exceeds the range. 

## 7.Technical Support

Email : support@paxini.com 

Tel : +86 0755-23574593 

Website : www.paxini.com 

## PaXini