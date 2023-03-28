import sys
import time
import pymongo
import requests
import json
import queue

agent = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36 Edg/99.0.1150.55'
headers = {'User-Agent': agent}

tags_url = lambda s: f'https://hub.docker.com/v2/repositories/{s}/tags/'
fd_url = lambda s: f'https://hub.docker.com/v2/repositories/{s}/'
cmd_url = lambda a, b: f'https://hub.docker.com/v2/repositories/{a}/tags/{b}/images/'
proxy_url = 'http://tiqu.pyhttp.taolop.com/getip?count=1&neek=28608&type=2&yys=0&port=1&sb=&mr=1&sep=0&ts=1&time=4'

mg_client = pymongo.MongoClient('mongodb://admin:UlnZ3lQEidijeF!@localhost:27017/')
db = mg_client['admin']
mycol = db['docker']
proxy_col = db['proxy']

ip = "27.200.90.62"
port = "9700"
username = "a5de8f81528680d7"
password = "8948534202074"

proxy = "http://%(ip)s:%(port)s" % {
    "ip": ip,
    "port": port,
}
proxies = {
    "http": proxy,
    "https": proxy,
}

proxies_list = []
proxies_num = 0

def get_full(data):
    global mycol, fd_url, headers, proxies
    id = data['id']
    name = data['name']
    fd = data['full_description']
    if fd == 'NULL':
        errnum = 0
        proxy_num = 0
        fdu = fd_url(name)
        while True:
            try:
                res = requests.get(fdu, headers=headers, proxies=proxies, timeout=20)
            except Exception as e:
                print(id, 'full err:', str(e))
                proxy_num += 1
                if proxy_num >= 3:
                    proxy_num = 0
                    update_proxy()
                    time.sleep(0.3)
                    continue
                errnum += 1
                if errnum >= 4:
                    return False
                continue
            if res.status_code == 200:
                json_result = json.loads(res.text)
                try:
                    full_description = json_result['full_description']
                except:
                    mycol.update_one({'id': id}, {'$set': {'full_description': 'NOTHING'}})
                    return True
                if full_description == '':
                    full_description = 'NOTHING'
                elif full_description is None:
                    full_description = 'NOTHING'
                else:
                    full_description = full_description.split('\t')
                    full_description = ''.join(full_description)
                    full_description = full_description.split('\u0026')
                    full_description = '&'.join(full_description)
                mycol.update_one({'id': id}, {'$set': {'full_description': full_description}})
                return True
            elif res.status_code == 404:
                full_description = '404'
                mycol.update_one({'id': id}, {'$set': {'full_description': full_description}})
                return True
            elif res.status_code == 429:
                print(id, name, 'full 429 sleep')
                time.sleep(5)
                continue
            else:
                errnum += 1
            if errnum >= 4:
                return False
    else:
        return True


def get_tages(data):
    global mycol, tags_url, headers, proxies
    id = data['id']
    name = data['name']
    tags_dict = {}
    errnum = 0
    tgu = tags_url(name)
    num = 0
    proxy_num = 0
    while True:
        try:
            res = requests.get(tgu, headers=headers, proxies=proxies, timeout=20)
        except Exception as e:
            print(id, 'tags err:', str(e))
            proxy_num += 1
            if proxy_num >= 3:
                proxy_num = 0
                update_proxy()
                time.sleep(0.3)
                continue
            errnum += 1
            if errnum >= 4:
                return -1, {}
            continue
        if res.status_code == 200:
            json_result = res.json()
            try:
                count = json_result['count']
            except:
                return -1, {}
            try:
                tags = json_result["results"]
            except:
                return -1, {}
            tags_num = len(tags)
            num += tags_num
            if tags_num == 0:
                return num, tags_dict
            for tag in tags:
                tagName = tag["name"]
                if tagName not in tags_dict:
                    tags_dict[tagName] = 1
            if num >= count:
                return num, tags_dict
            else:
                tgu = json_result['next']
        elif res.status_code == 404:
            return 0, {}
        elif res.status_code == 429:
            print(id, name, 'tag 429 sleep')
            time.sleep(5)
            continue
        else:
            errnum += 1
        if errnum >= 4:
            return -1, {}


def get_cmd(name, tages_name):
    global mycol, cmd_url, headers, proxies
    errnum = 0
    cmdu = cmd_url(name, tages_name)
    proxy_num = 0
    while True:
        try:
            res = requests.get(cmdu, headers=headers, proxies=proxies, timeout=20)
        except Exception as e:
            print(id, 'cmd err:', name, tages_name, str(e))
            proxy_num += 1
            if proxy_num >= 3:
                proxy_num = 0
                update_proxy()
                time.sleep(0.3)
                continue
            errnum += 1
            if errnum >= 4:
                return True, []
            continue
        if res.status_code == 200:
            if len(res.text) == 0:
                return True, []
            content = res.json()
            if content is None or len(content) == 0:
                return True, []
            try:
                imageHistory = []
                for commands in content[0]["layers"]:
                    instruction = str(commands["instruction"]).split('\t')
                    instruction = ''.join(instruction)
                    instruction = instruction.split('\u0026')
                    instruction = '&'.join(instruction)
                    imageHistory.append(instruction)  # .replace("\u0026","&").replace("\\", ""))
                return True, imageHistory
            except Exception as e:
                # print(id, name, tages_name, str(e))
                # print(res.text)
                return True, []
        elif res.status_code == 404:
            return True, []
        elif res.status_code == 429:
            print(id, name, tags_name, 'cmd 429 sleep')
            time.sleep(5)
            continue
        else:
            print('else', res.status_code, res.text, id, name, tages_name)
            errnum += 1
        if errnum >= 4:
            return True, []


id_0 = int(sys.argv[1])
id_1 = int(sys.argv[2])
myquery = {'id': {'$gte': id_0, '$lt': id_1}}
data_q = queue.Queue()
while True:
    all_data = mycol.find(myquery)
    try:
        for data in all_data:
            data_q.put(data)
        break
    except:
        continue
update_proxy()
while not data_q.empty():
    print('Remaining:', data_q.qsize())
    data = data_q.get()
    id = data['id']
    name = data['name']
    if not get_full(data):
        data_q.put(data)
        print(id, name, 'get_full failed')
        continue
    tags_num = int(data['tags_num'])
    if tags_num == -1:
        tn, tags_dict = get_tages(data)
        if tn == -1:
            data_q.put(data)
            print(id, name, 'get_tages failed')
            continue
        elif tn == 0:
            mycol.update_one({'id': id}, {'$set': {'tags_num': 0, 'tags': 'NOTHING'}})
            print(id, name, '0 tages success')
        else:
            output_dict = {}
            dict_num = 0
            mycol.update_one({'id': id}, {'$set': {'tags_num': tn, 'tags': str(list(tags_dict.keys()))}})
            for tags_name in tags_dict.keys():
                cmd_ok, cmd_list = get_cmd(name, tags_name)
                if not cmd_ok:
                    data_q.put(data)
                    print(id, name, tags_name, 'get_cmd failed')
                    continue
                else:
                    dict_num += 1
                    if len(cmd_list) == 0:
                        tags_dict[tags_name] = 'NOTHING'
                    else:
                        tags_dict[tags_name] = str(cmd_list)
                    output_dict[f'tags_{str(dict_num)}'] = '{\'' + tags_name + '\':' + tags_dict[tags_name] + '}'
            mycol.update_one({'id': id}, {'$set': output_dict})
            print(id, name, tn, 'success')
