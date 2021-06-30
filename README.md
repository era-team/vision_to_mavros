A collection of [ROS](https://www.ros.org/) and non-ROS (Python, cpp) code that converts data from vision-based system (external localization system like fiducial tags, VIO, SLAM, or depth image) to corresponding [MAVROS](http://wiki.ros.org/mavros) topics or [MAVLink](https://mavlink.io/en/) messages that can be consumed by a flight control stack to support precise localization and navigation tasks.

The code has been tested and come with instructions to be used with [ArduPilot](https://ardupilot.org/). The main sensor is the [Intel Realsense Tracking camera T265](https://www.intelrealsense.com/tracking-camera-t265/).

---

# Ссылка на базовый репозиторий [Vision to MAVROS](https://github.com/thien94/vision_to_mavros)

---

## Важно
! Так как используемый подход подразумевает использование ROS, то необходимо его предварительно установить на компьютер компаньен. Как установить ROS [читайте на оффициальной странице](http://wiki.ros.org/Installation/UbuntuARM).

! Необходимо учитывать, что одометрия передается камерой T265, соответственно неоходимо [установить SDK для RealSence](https://github.com/IntelRealSense/librealsense/blob/master/doc/installation_jetson.md).

---

Основной смысл того, что необходимо сделать чтобы передать одометрию от камеры T265 в полетный контроллер PX4 с прошивкой от Ardupilot, описан в статье [Non-GPS Navigation](https://ardupilot.org/copter/docs/common-vio-tracking-camera.html) на сайте Ardupilot. Существует несколько подходов для достижения этой цели. Оба они основываются на том, чтобы плучить положение камеры в пространстве, преобразовать его и передать в полетный контроллер, чаще всего по протоколу MAVLINK.
В статье описаны два подхода: через ROS и non-Ros.
- non-Ros метод выполняет преобразования python скриптом и должен быть запущен в главной программе или параллельно с ней.
- ROS метод основан на использовании одноименной системы. MAVROS модуль используется для частичного преобразования положения камеры и передачи его в полетный контроллер.

Для более глубокого понимания как это устроено и работает можно изучить [GSoC 2019: Integration of ArduPilot and VIO tracking camera for GPS-less localization and navigation](https://discuss.ardupilot.org/t/gsoc-2019-integration-of-ardupilot-and-vio-tracking-camera-for-gps-less-localization-and-navigation/42394).

Данный репозиторий содержит сконфигурированный узел для ROS. Настройка произведена для полетного контроллера PX4 с прошивкой Ardupilot (v 4.0.7) подключенного к Jetson Nano по UART.

- Jetson Nano J41 Pin 8 (TXD) → PX4 Telem RXD
- Jetson Nano J41 Pin 10 (RXD) → PX4 Telem RXD TXD
- Jetson Nano J4 Pin 6 (GND) → PX4 Telem RXD GND

Болле подробно о Jetson Nano UART можно прочитать на [JetsonHacks](https://www.jetsonhacks.com/2019/10/10/jetson-nano-uart/)

Вся методика подключения и настройка взята их этого описания [Non-GPS Navigation](https://discuss.ardupilot.org/t/integration-of-ardupilot-and-vio-tracking-camera-part-2-complete-installation-and-indoor-non-gps-flights/43405). 


В репозитории содержится [конфигурационный файл]() для PX4 в папке [PX4Param](), который необходимо загрузить на полетный контроллер для корректной работы одометрии и полетного контроллера. Основные натройки:

- Квадрокоптер с рамой V типа.
- Плата питания Holibro PM07.
- Выключен компасс.
- Подключена радиотелеметрия.
- Настроено соединение с компьютером компаньеном (Jetson Nano).
- Включено получение внешней одометрии.

Конфигурация PX4 взята так же из статьи [Non-GPS Navigation ](https://discuss.ardupilot.org/t/integration-of-ardupilot-and-vio-tracking-camera-part-2-complete-installation-and-indoor-non-gps-flights/43405). При необходимости можно настроить PX4 самостоятельно.

---

## Перед запуском
Предварительно необходимо настроить время на компьютере компаньене если это необходимо. Например, в Jetson nano отсутствует батарейка для часов реального времени и после перезапуска происходит сброс временных настроек.

Необходимо клонировать репозиторий в рабочее пространство ROS и пересебрать его. После сборки в ROS должен увидеть пакет [vision_to_mavros]().

## Запуск
Для начала трансляции одометрии в полетный контроллер необходимо запустить [vision_to_mavros]() с помошью launch-файла [t265_all_nodes.launch](). Данный файл имеет несколько настроек:

* run_camera

        type: bool
        default: false
        Необходимо ли стартовать узел realsence

* fcu_url

        type: string
        default: /dev/ttyTHS1:921600
        По умолчанию настроен для подключения к PX4 с Ardupilot через UART на скорости 921600 бод.

* source_frame_id
        type: string
        default: /t265_link
        Система координат из которой происходит трансформация (Ros doc: The frame where the data originated)

* target_frame_id
        type: string
        default: /t265_link
        Система координат в которую производим трансформацию (Ros doc: The frame to which data should be transformed)

### Замечание
Необходимо учитывать ориентацию камеры на дроне. Вращение каммеры происходит с помощью парметров в файле [t265_tf_to_mavros.launch](). Настройка по умолчанию падразумевает что камера смотрит под углом 45 градусов к плоскости горизонта назад по ходу дрона.

        <param name="roll_cam"          value="0" />
        <param name="pitch_cam"         value="0.785" />
        <param name="yaw_cam"           value="3.14" />
        <param name="gamma_world"       value="1.5707963" />

Более подробно об ориентации камеры можно прочитать в статье [Integration of ArduPilot and VIO tracking camera](https://discuss.ardupilot.org/t/integration-of-ardupilot-and-vio-tracking-camera-part-2-complete-installation-and-indoor-non-gps-flights/43405).

---

# Основные осправления
- Добавлены параметры в [t265_all_nodes.launch]()
- Допавлен файл параметров для PX4 [PX4Param]()
- Отключена сборка файла t265_fisheye_undistort.cpp