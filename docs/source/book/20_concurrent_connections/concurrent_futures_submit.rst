Метод submit и работа с futures
-------------------------------

Метод submit отличается от метода map:

* submit запускает в потоке только одну функцию
* с помощью submit можно запускать разные функции с разными несвязанными аргументами,
  а map надо обязательно запускать с итерируемым объектами в роли аргументов
* submit сразу возвращает результат, не дожидаясь выполнения функции
* submit возвращает специальный объект Future, который представляет
  выполнение функции.

  * submit возвращает Future для того чтобы вызов submit не блокировал
    код. Как только submit вернул Future, код может выполняться дальше.
    И как только запущены все функции в потоках, можно начинать запрашивать
    Future о том готовы ли результаты. Или воспользоваться специальной функцией
    as_completed, которая сама запрашивает результат, а код получает его
    по мере готовности

* submit возвращает результаты в порядке готовности, а не в порядке аргументов
* submit можно передавать ключевые аргументы, а map только позиционные

Метод submit использует объект `Future <https://en.wikipedia.org/wiki/Futures_and_promises>`__ - это
объект, который представляет отложенное вычисление. Этот объект можно
запрашивать о состоянии (завершена работа или нет), можно получать
результаты или исключения, которые возникли в процессе работы, по мере
возникновения.
Future не нужно создавать вручную, эти объекты создаются методом submit.


Пример запуска функции в потоках с помощью submit (файл
netmiko_threads_submit_basics.py):

.. literalinclude:: /pyneng-examples-exercises/examples/20_concurrent_connections/netmiko_threads_submit_basics.py
  :language: python
  :linenos:


Остальной код не изменился, поэтому разобраться надо только с блоком,
который запускает функцию send_show в потоках:

.. code:: python

    with ThreadPoolExecutor(max_workers=2) as executor:
        future_list = []
        for device in devices:
            future = executor.submit(send_show, device, 'sh clock')
            future_list.append(future)
        for f in as_completed(future_list):
            print(f.result())

Теперь в блоке with два цикла: 

* ``future_list`` - это список объектов future:

  * для создания future используется функция submit 
  * ей как аргументы передаются: имя функции, которую надо выполнить, и ее аргументы 

* следующий цикл проходится по списку future с помощью функции as_completed. Эта функция
  возвращает future только когда они завершили работу или были отменены.
  При этом future возвращаются по мере завершения работы, не в порядке добавления в
  список future_list


.. note::

    Создание списка с future можно сделать с помощью list comprehensions:
    ``future_list = [executor.submit(send_show, device, 'sh clock') for device in devices]``

Результат выполнения:

::

    $ python netmiko_threads_submit_basics.py
    ThreadPoolExecutor-0_0 root INFO: ===> 17:32:59.088025 Connection: 192.168.100.1
    ThreadPoolExecutor-0_1 root INFO: ===> 17:32:59.094103 Connection: 192.168.100.2
    ThreadPoolExecutor-0_1 root INFO: <=== 17:33:11.639672 Received: 192.168.100.2
    {'192.168.100.2': '*17:33:11.429 UTC Thu Jul 4 2019'}
    ThreadPoolExecutor-0_1 root INFO: ===> 17:33:11.849132 Connection: 192.168.100.3
    ThreadPoolExecutor-0_0 root INFO: <=== 17:33:17.735761 Received: 192.168.100.1
    {'192.168.100.1': '*17:33:17.694 UTC Thu Jul 4 2019'}
    ThreadPoolExecutor-0_1 root INFO: <=== 17:33:23.230123 Received: 192.168.100.3
    {'192.168.100.3': '*17:33:23.188 UTC Thu Jul 4 2019'}


Обратите внимание, что порядок не сохраняется и зависит от того, какие
функции раньше завершили работу.

Future
~~~~~~

.. code:: python
    In [4]: f1 = executor.submit(send_show, r1, 'sh clock')
       ...: f2 = executor.submit(send_show, r2, 'sh clock')
       ...: f3 = executor.submit(send_show, r3, 'sh clock')
       ...:
    ThreadPoolExecutor-0_0 root INFO: ===> 17:53:19.656867 Connection: 192.168.100.1
    ThreadPoolExecutor-0_1 root INFO: ===> 17:53:19.657252 Connection: 192.168.100.2

    In [5]: print(f1, f2, f3, sep='\n')
    <Future at 0xb488e2ac state=running>
    <Future at 0xb488ef2c state=running>
    <Future at 0xb488e72c state=pending>

    ThreadPoolExecutor-0_1 root INFO: <=== 17:53:25.757704 Received: 192.168.100.2
    ThreadPoolExecutor-0_1 root INFO: ===> 17:53:25.869368 Connection: 192.168.100.3

    In [6]: print(f1, f2, f3, sep='\n')
    <Future at 0xb488e2ac state=running>
    <Future at 0xb488ef2c state=finished returned dict>
    <Future at 0xb488e72c state=running>

    ThreadPoolExecutor-0_0 root INFO: <=== 17:53:30.431207 Received: 192.168.100.1
    ThreadPoolExecutor-0_1 root INFO: <=== 17:53:31.636523 Received: 192.168.100.3

    In [7]: print(f1, f2, f3, sep='\n')
    <Future at 0xb488e2ac state=finished returned dict>
    <Future at 0xb488ef2c state=finished returned dict>
    <Future at 0xb488e72c state=finished returned dict>


Чтобы посмотреть на future, в скрипт добавлены несколько строк с выводом
информации (netmiko_threads_submit_verbose.py):

.. code:: python

    from concurrent.futures import ThreadPoolExecutor, as_completed
    from pprint import pprint
    from datetime import datetime
    import time

    import yaml
    from netmiko import ConnectHandler


    start_msg = '===> {} Connection to device: {}'
    received_msg = '<=== {} Received result from device: {}'


    def connect_ssh(device_dict, command):
        print(start_msg.format(datetime.now().time(), device_dict['ip']))
        if device_dict['ip'] == '192.168.100.1':
            time.sleep(10)
        with ConnectHandler(**device_dict) as ssh:
            ssh.enable()
            result = ssh.send_command(command)
            print(received_msg.format(datetime.now().time(), device_dict['ip']))
        return {device_dict['ip']: result}


    def threads_conn(function, devices, limit=2, command=''):
        all_results = {}
        with ThreadPoolExecutor(max_workers=limit) as executor:
            future_ssh = []
            for device in devices:
                future = executor.submit(function, device, command)
                future_ssh.append(future)
                print('Future: {} for device {}'.format(future, device['ip']))
            for f in as_completed(future_ssh):
                result = f.result()
                print('Future done {}'.format(f))
                all_results.update(result)
        return all_results


    if __name__ == '__main__':
        devices = yaml.load(open('devices.yaml'))
        all_done = threads_conn(connect_ssh,
                                devices['routers'],
                                command='sh clock')
        pprint(all_done)

    Так как в прошлом варианте мы уже проверили, что результат
    возвращается в порядке выполнения, тут функция threads_conn
    возвращает словарь, а не список.

Результат выполнения:

::

    $ python netmiko_threads_submit_verbose.py
    ===> 06:16:56.059256 Connection to device: 192.168.100.1
    Future: <Future at 0xb68427cc state=running> for device 192.168.100.1
    ===> 06:16:56.059434 Connection to device: 192.168.100.2
    Future: <Future at 0xb68483ac state=running> for device 192.168.100.2
    Future: <Future at 0xb6848b4c state=pending> for device 192.168.100.3
    <=== 06:17:01.482761 Received result from device: 192.168.100.2
    ===> 06:17:01.589605 Connection to device: 192.168.100.3
    Future done <Future at 0xb68483ac state=finished returned dict>
    <=== 06:17:07.226815 Received result from device: 192.168.100.3
    Future done <Future at 0xb6848b4c state=finished returned dict>
    <=== 06:17:11.444831 Received result from device: 192.168.100.1
    Future done <Future at 0xb68427cc state=finished returned dict>
    {'192.168.100.1': '*06:17:11.273 UTC Mon Aug 28 2017',
     '192.168.100.2': '*06:17:01.310 UTC Mon Aug 28 2017',
     '192.168.100.3': '*06:17:07.055 UTC Mon Aug 28 2017'}

Так как по умолчанию используется ограничение в два потока, только два
из трех future показывают статус running. Третий находится в состоянии
pending и ждет, пока до него дойдет очередь.

Обработка исключений
~~~~~~~~~~~~~~~~~~~~

Если при выполнении функции возникло исключение, оно будет сгенерировано
при получении результата

Например, в файле devices.yaml пароль для устройства 192.168.100.2
изменен на неправильный:

::

    $ python netmiko_threads_submit.py
    ===> 06:29:40.871851 Connection to device: 192.168.100.1
    ===> 06:29:40.872888 Connection to device: 192.168.100.2
    ===> 06:29:43.571296 Connection to device: 192.168.100.3
    <=== 06:29:48.921702 Received result from device: 192.168.100.3
    <=== 06:29:56.269284 Received result from device: 192.168.100.1
    Traceback (most recent call last):
    ...
      File "/home/vagrant/venv/py3_convert/lib/python3.6/site-packages/netmiko/base_connection.py", line 500, in establish_connection
        raise NetMikoAuthenticationException(msg)
    netmiko.ssh_exception.NetMikoAuthenticationException: Authentication failure: unable to connect cisco_ios 192.168.100.2:22
    Authentication failed.

Так как исключение возникает при получении результата, легко добавить
обработку исключений (файл netmiko_threads_submit_exception.py):

.. code:: python

    from concurrent.futures import ThreadPoolExecutor, as_completed
    from pprint import pprint
    from datetime import datetime
    import time

    import yaml
    from netmiko import ConnectHandler
    from netmiko.ssh_exception import NetMikoAuthenticationException


    start_msg = '===> {} Connection to device: {}'
    received_msg = '<=== {} Received result from device: {}'


    def connect_ssh(device_dict, command):
        print(start_msg.format(datetime.now().time(), device_dict['ip']))
        if device_dict['ip'] == '192.168.100.1':
            time.sleep(10)
        with ConnectHandler(**device_dict) as ssh:
            ssh.enable()
            result = ssh.send_command(command)
            print(received_msg.format(datetime.now().time(), device_dict['ip']))
        return {device_dict['ip']: result}


    def threads_conn(function, devices, limit=2, command=''):
        all_results = {}
        with ThreadPoolExecutor(max_workers=limit) as executor:
            future_ssh = [executor.submit(function, device, command)
                          for device in devices]
            for f in as_completed(future_ssh):
                try:
                    result = f.result()
                except NetMikoAuthenticationException as e:
                    print(e)
                else:
                    all_results.update(result)
        return all_results


    if __name__ == '__main__':
        devices = yaml.load(open('devices.yaml'))
        all_done = threads_conn(connect_ssh,
                                devices['routers'],
                                command='sh clock')
        pprint(all_done)

Результат выполнения:

::

    $ python netmiko_threads_submit_exception.py
    ===> 06:45:56.327892 Connection to device: 192.168.100.1
    ===> 06:45:56.328190 Connection to device: 192.168.100.2
    ===> 06:45:58.964806 Connection to device: 192.168.100.3
    Authentication failure: unable to connect cisco_ios 192.168.100.2:22
    Authentication failed.
    <=== 06:46:04.325812 Received result from device: 192.168.100.3
    <=== 06:46:11.731541 Received result from device: 192.168.100.1
    {'192.168.100.1': '*06:46:11.556 UTC Mon Aug 28 2017',
     '192.168.100.3': '*06:46:04.154 UTC Mon Aug 28 2017'}

Конечно, обработка исключения может выполняться и внутри функции
connect_ssh, но это просто пример того, как можно работать с
исключениями при использовании future.
