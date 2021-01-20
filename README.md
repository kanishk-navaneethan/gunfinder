@@ -6,6 +6,8 @@
import pytesseract
import difflib
import json
import nltk
import requests

pytesseract.pytesseract.tesseract_cmd = 'C:/Program Files (x86)/Tesseract-OCR/tesseract'
last_time = time.time()
@@ -17,7 +19,11 @@
data = {
    "users": []
}
death_words = ['headshot', 'Playzone']

killed = []
killed_list = []

death_words = ['headshot', 'Playzone', 'falling']
gun_list = ['AKM','AUG A3','Beryl M762','Groza','M16A4','M416','Mk47 Mutant','QBZ95','SCAR-L','G36C', # Assault Rifles
            'Mini 14','Mk14 EBR','QBU','SKS','SLR','VSS', # DMRs
            'Winchester', # Rifles
@@ -44,37 +50,178 @@ def LD(s, t):
    res = min([LD(s[:-1], t)+1,
               LD(s, t[:-1])+1, 
               LD(s[:-1], t[:-1]) + cost])

    return res


def user_exists(user):
    print('--------------------------------- CHECK USER EXISTS ---------------------------------')
    print('Checking user ' + user)    
    has_player = False
    for player in data['users']:
        if LD(user, player['name']) < 2:
            has_player = True
            break

    print(has_player)
    return has_player
        if nltk.edit_distance(user, player['name']) <= 2:
            return True
    return False

def weapon_exists(weapon):
    has_weapon = False

    return has_weapon

def get_user_from_list(name):
    print('--------------------------------- GET USER FROM LIST ---------------------------------')
    for n in data['users']:
    for player in data['users']:        
        # if new name is close enough to previous name, update and return
        if LD(name, n['name']) < 2:
            n['name'] = name
            return n
        if nltk.edit_distance(name, player['name']) <= 2:
            player['name'] = name
            return player


def has_word(needle, word, haystack):
    return needle in difflib.get_close_matches(word, haystack)


def get_weapon(words):
    weapon = words[len(words) - 1]
    possible_weapons = difflib.get_close_matches(weapon, gun_list)
    if len(possible_weapons) > 0 and nltk.edit_distance(weapon, possible_weapons[0]) <= 2:
        weapon = possible_weapons[0]
        return weapon
    return ''


def is_killed(name):
    for killed_player in killed:
        if nltk.edit_distance(killed_player, name) < 2:
            return True
    return False

def add_new_user(index, words):
    name = words[index - 1]
    if is_killed(name) == False and len(name) > 4:
        weapon = get_weapon(words)        
        user = {
            'name': name,
            'weapon_1': weapon,
            'weapon_2': '',
            'handgun': '',
            'kills': 1,
            'temp_weapon': ''
        }
        data['users'].append(user)
        return user

def update_user(index, words):
    weapon = get_weapon(words)
    user = get_user_from_list(words[index - 1])
    if user != None:
        if user['weapon_1'] != weapon and user['weapon_2'] != weapon:
            if user['weapon_1'] == '':
                user['weapon_1'] = weapon
            else:
                if user['weapon_2'] == '':
                    user['weapon_2'] = weapon
                else:
                    if weapon == user['temp_weapon']:
                        user['weapon_2'] = user['temp_weapon']
                        user['temp_weapon'] = ''
                    else:
                        user['temp_weapon'] = user['weapon_1']
                        user['weapon_1'] = weapon
        if len(words) > index + 1 and is_killed(words[index + 1]) == False:
            user['kills'] += 1
        print(get_user_from_list(words[index - 1]))
    else:
        # Could not find user
        user = user

def remove_user(name):
    print('----------------------------------------REMOVE USER ' + name)
    for i in range(len(data['users'])): 
        if nltk.edit_distance(name, data['users'][i]['name']) < 2 and len(name) > 4:
            killed.append(data['users'][i]['name'])
            killed_list.append(data['users'][i])
            del data['users'][i]
            break

def check_killed_users(index, words):
    if len(words) > index + 1 and user_exists(words[index + 1]) == True:
        # killed user exists in the array of users
        remove_user(words[index + 1])
    else :
        # just add killed user to list of killed
        print('testing error')
        print(words)
        if index != 0:
            if len(words) > index + 1 and is_killed(words[index + 1]) == False:
                killed.append(words[index + 1])
        else:
            if is_killed(words[index]) == False:
                killed.append(words[index])

def parse_line(index, words):    
    # If index >= 1, it means 'killed' was at 2nd index, which means that the username is on the first index
    if index >= 1 and user_exists(words[index - 1]) == False:
        # if not in list - add a new user with all their info
        print('--------------------------------- ADDING USER ' + words[index - 1])
        user = add_new_user(index, words)
        check_killed_users(index, words)
    else:
        if index >= 1:
            print('--------------------------------- UPDATING USER ' + words[index - 1])
            # Check the info of the words and see if guns have updated
            user = update_user(index, words)
            check_killed_users(index, words)
        else:
            # could not find first user so we cant update him, however we can remove a user if found
            check_killed_users(index, words)

def other_death_kill(words):
    if has_word('Playzone', words[len(words) - 1], death_words):
        return True
    if has_word('falling', words[len(words) - 1], death_words):
        return True        
    return False

print(nltk.edit_distance("humpty", "dumpty"))

def api_request():
    url = "https://api.pubg.com/shards/steam/players?filter[playerNames]=Ruru_Zero"

    header = {
        "Authorization": "Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJqdGkiOiJhZjQxMDZkMC0xZmRlLTAxMzctNWMwNS0xZDI1YjRjNGIzZGEiLCJpc3MiOiJnYW1lbG9ja2VyIiwiaWF0IjoxNTUxNjE2NTAwLCJwdWIiOiJibHVlaG9sZSIsInRpdGxlIjoicHViZyIsImFwcCI6Imd1bmZpbmRlciJ9.bfYewLKMtWYLAanOQiW136QHJj_1zo3e5GWACN9ed64",
        "Accept": "application/vnd.api+json"
    }

    r = requests.get(url, headers=header)


    print(r.text)

    url = "https://api.pubg.com/shards/steam/matches/d262b909-c994-4b85-a51d-9f76d6ad49ae"

    header = {
        "Authorization": "Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJqdGkiOiJhZjQxMDZkMC0xZmRlLTAxMzctNWMwNS0xZDI1YjRjNGIzZGEiLCJpc3MiOiJnYW1lbG9ja2VyIiwiaWF0IjoxNTUxNjE2NTAwLCJwdWIiOiJibHVlaG9sZSIsInRpdGxlIjoicHViZyIsImFwcCI6Imd1bmZpbmRlciJ9.bfYewLKMtWYLAanOQiW136QHJj_1zo3e5GWACN9ed64",
        "Accept": "application/vnd.api+json"
    }

    r = requests.get(url, headers=header)


    print(r.text)

    url = "https://api.pubg.com/shards/steam/players?filter[playerIds]=8a7a70c0-cbba-4b08-8690-0ff55b06a75b"

    header = {
        "Authorization": "Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJqdGkiOiJhZjQxMDZkMC0xZmRlLTAxMzctNWMwNS0xZDI1YjRjNGIzZGEiLCJpc3MiOiJnYW1lbG9ja2VyIiwiaWF0IjoxNTUxNjE2NTAwLCJwdWIiOiJibHVlaG9sZSIsInRpdGxlIjoicHViZyIsImFwcCI6Imd1bmZpbmRlciJ9.bfYewLKMtWYLAanOQiW136QHJj_1zo3e5GWACN9ed64",
        "Accept": "application/vnd.api+json"
    }

    r = requests.get(url, headers=header)


    print(r.text)

api_request()
while(True):
    screen = ig.grab(bbox=(1200,70,1920,300))
    # screen = ig.grab(bbox=(1200,60,1920,300))
    screen = ig.grab(bbox=(1200,60,1920,300))
    #print('Loop took {} seconds',format(time.time()-last_time))
    gray = cv2.cvtColor(np.array(screen), cv2.COLOR_BGR2GRAY)    
    # blurred = cv2.blur(gray, (3,3))
@@ -86,64 +233,27 @@ def get_user_from_list(name):
        text_chunk = output.split('\n')
        for text_line in text_chunk:
            words = text_line.split(' ')
            print('------------------ NEW WORDS LINE --------------------')
            print(words)
            for index, word in enumerate(words):
                print(difflib.get_close_matches(word, key_words))
                if 'killed' in difflib.get_close_matches(word, key_words):
                    # check if user already in list
                    # x killed y with z
                    if index > 0 and user_exists(words[index - 1]) == False:
                        # if not in list - add a new user with all their info
                        # TODO: Check headshot killfeed
                        weapon = ''
                        if len(words) >= index + 3 and 'with' in difflib.get_close_matches(words[index + 2], key_words):
                            weapon = words[index + 3]

                        print('----------------------------------------WEAPON CHECK ')
                        print(difflib.get_close_matches(weapon, gun_list))
                        if weapon != '':
                            possible_weapons = difflib.get_close_matches(weapon, gun_list)
                            if len(possible_weapons) > 0:
                                weapon = possible_weapons[0]
                        user = {
                            'name': words[index - 1],
                            'weapon_1': weapon,
                            'weapon_2': '',
                            'handgun': '',
                            'kills': 1,
                            'temp_weapon': ''
                        }
                        data['users'].append(user)
                        print(words)
                    else:
                        # Check the info of the words and see if guns have updated
                        print('User exists')
                        weapon = ''
                        if len(words) >= index + 3 and 'with' in difflib.get_close_matches(words[index + 2], key_words):
                            weapon = words[index + 3]
                        if weapon != '':
                            possible_weapons = difflib.get_close_matches(weapon, gun_list)
                            if len(possible_weapons) > 0:
                                weapon = possible_weapons[0]

                        user = get_user_from_list(words[index - 1])
                        if user != None:
                            if user['weapon_1'] != weapon and user['weapon_2'] != weapon:
                                if user['weapon_2'] == '':
                                    user['weapon_2'] = weapon
                                else:
                                    if weapon == user['temp_weapon']:
                                        user['weapon_2'] = user['temp_weapon']
                                        user['temp_weapon'] = ''
                                    else:
                                        user['temp_weapon'] = user['weapon_1']
                                        user['weapon_1'] = weapon

                            print(get_user_from_list(words[index - 1]))


        print('users:')
            if other_death_kill(words):
                # remove user to killed
                print('--------------- DIED TO PLAYZONE ------------------')
                check_killed_users(0, words)
            else:
                for index, word in enumerate(words):
                    print(difflib.get_close_matches(word, key_words))                
                    # Checks if 'word' is in 'key_words' and checks if it is equal to 'killed'
                    if has_word('killed', word, key_words):
                        # parse the line (index should be = 1, since x killed y)
                        parse_line(index, words)
                        # break since we found 'killed' and already parsed entire line
                        break
        print('---- CURRENT ALIVE USERS ----')
        print(data['users'])
        print('---- CURRENT KILLED USERS ----')
        print(killed_list)
        print('---- CURRENT KILLED USER NAMES ----')
        print(killed)
        # print(output)
    last_time = time.time()
    if cv2.waitKey(25) & 0xFF == ord('q'):
