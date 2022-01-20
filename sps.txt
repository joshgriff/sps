try:
	import numpy as np
except:
	print('Optional numpy not installed!')
try:
	import matplotlib.pyplot as plt
except:
	print('Optional matplotlib not installed!')
import requests
from json import JSONDecoder

# stat_api = 'https://game-api.splinterlands.com/cards/get_details'

# s = requests.get(stat_api).json()

# for a given mana level,

# with a given set of playable cards

# compute the win probabilities of each summoner

# and optimal card placement

deck = ['','','','','','','','']


class card_details:

	def __init__(self):
		self.r = requests.get('https://api2.splinterlands.com/cards/get_details').json()
		self.cd = {}
		for i in range(len(self.r)):
			self.cd[str(self.r[i]['id'])] = self.r[i]

	def get(self,id,field):
		return(self.cd[str(id)][str(field)])

CD = card_details()

def id_to_name(id):

	return(CD.get(id,'name'))




# def cget(id,field):




def get_history(player):

	return(requests.get('https://api.splinterlands.io/battle/history?player='+player).json()['battles'])

# Iterate available decks 

# look at prior matches

# compute opponent stat correlation with outcome

def hstat(player):

	bs = get_history(player)	
	return(bs)

def plt_rating(h):

	r = [int(hh['player_1_rating_final'])-int(hh['player_1_rating_initial']) for hh in h]
	r = np.array(r)
	r -= r.min()
	r = r/r.max()
	r -= r.mean()
	r = r/r.std()
	plt.figure('plt_all')
	# plt.cla()
	plt.plot(r)
	plt.show(block=False)
	# plt.pause(.0001)

def plt_dpt(h):
	
	jd = JSONDecoder()
	dpts = []
	for hh in h:
		
		ts = jd.decode(hh['details'])

		if 'rounds' not in str(ts.keys()):
			dpts.append(0)

		else:
			ts = ts['rounds']

			ds = []
			i = 0
			for t in ts:
				for a in t['actions']:
					if 'damage' in  str(a.keys()):
						i += 1
						ds.append(np.float(a['damage']))
			if i:
				dpts.append(np.sum(ds)/i)
			else:
				dpts.append(0)

	dpts = np.array(dpts)
	dpts -= dpts.min()
	dpts = dpts/dpts.max()
	dpts -= dpts.mean()
	dpts = dpts/dpts.std()
	plt.figure('plt_all')
	# plt.cla()
	plt.plot(dpts)
	plt.show(block=False)
	# plt.pause(.0001)

def dpt_r_corr(h):

	jd = JSONDecoder()
	dpts = []
	for hh in h:
		
		ts = jd.decode(hh['details'])

		if 'rounds' not in str(ts.keys()):
			dpts.append(0)

		else:
			ts = ts['rounds']

			ds = []
			i = 0
			for t in ts:
				for a in t['actions']:
					if 'damage' in  str(a.keys()):
						i += 1
						ds.append(np.float(a['damage']))
			if i:
				dpts.append(np.sum(ds)/i)
			else:
				dpts.append(0)

	dpts = np.array(dpts)
	dpts -= dpts.min()
	dpts = dpts/dpts.max()
	dpts -= dpts.mean()
	dpts = dpts/dpts.std()

	r = [int(hh['player_1_rating_final'])-int(hh['player_1_rating_initial']) for hh in h]
	r = np.array(r)
	r -= r.min()
	r = r/r.max()
	r -= r.mean()
	r = r/r.std()

	print('dpt X rating corr:'+str(r@dpts))

def get_collection(player):
	c = requests.get('https://api.splinterlands.io/cards/collection/'+player).json()
	return(c)

def get_card_info(card_id):
	c = requests.get('https://api.splinterlands.io/cards/find?ids='+card_id)
	return(c)

def best_plays(mq,pn='goatsie',rate_enemy_comp=False):

	# pn='goatsie'
	h = get_history(pn)

	jd = JSONDecoder()

	gd_tbl = []

	for hh in h:
		
		mana = hh['mana_cap']

		gd = jd.decode(hh['details'])

		if 'type' not in gd.keys():

			if gd['team1']['player'] == pn:
				if rate_enemy_comp:
					team = gd['team2']
				else:
					team = gd['team1']
			else:
				if rate_enemy_comp:
					team = gd['team1']
				else:
					team = gd['team2']

			for m in team['monsters']:

				if hh['winner'] == pn:

					gd_tbl.append([m['card_detail_id'],mana,1])

				else:

					gd_tbl.append([m['card_detail_id'],mana,0])


	m0 = '100'
	bb = {m0:{}}

	for i in range(len(gd_tbl)):

		mm = str(gd_tbl[i][1])
		if mm not in bb.keys():
			bb[mm] = {}
			
		mn = id_to_name(str(gd_tbl[i][0]))
		if mn in bb[mm].keys():

			bb[mm][mn]['w'] += gd_tbl[i][2]
			bb[mm][mn]['l'] += 1-gd_tbl[i][2]
		else:

			bb[mm][mn] = {'w':gd_tbl[i][2],'l':1-gd_tbl[i][2]}

		if mn in bb[m0].keys():
			bb[m0][mn]['w'] += gd_tbl[i][2]
			bb[m0][mn]['l'] += 1-gd_tbl[i][2]
		else:
			bb[m0][mn] = {'w':gd_tbl[i][2],'l':1-gd_tbl[i][2]}

	print('Wins / Losses While Playing Monster at Mana Cap')
	print(30*' '+'  mana   | total')
	print(30*' '+'   w/l   | w/l')

	# for km in bb.keys():
	km = str(mq)
	for kmm in bb[km].keys():
		kmm2 = kmm+(30-len(kmm))*' '
		# print(kmm2+' : '+str(bb[km][kmm]['w'])+' ; '+str(bb[km][kmm]['l'])+' ; '+str(bb[m0][kmm]['w'])+' ; '+str(bb[m0][kmm]['l']))
		print(kmm2+' | '+
			str(bb[km][kmm]['w'])+' ; '+str(bb[km][kmm]['l'])+' | '+
			str(bb[m0][kmm]['w'])+' ; '+str(bb[m0][kmm]['l']))


def runtime():

	while 1:

		print('Player Name')
		pn = input()

		print('~\n~\nMana')
		mq = input()

		print('~\n~\n')

		print('Plays against')

		best_plays(mq,pn,rate_enemy_comp=True)

		print('~\n~\n')		

		print('Plays as')

		best_plays(mq,pn,rate_enemy_comp=False)

		print('~\n~\n')		



runtime()




# for km in bb.keys():
# 	for kmm in bb[km].keys():
# 		bb[km][kmm]['r']=bb[km][kmm]['w']/(bb[km][kmm]['w']+bb[km][kmm]['l'])

# return(bb)

# pn = 'goatsie'
# pn = 'thecryptonnecter'
# pn = 'liuzz537'

# print('Player Name: '+pn)
# h = get_history(pn)

# jd = JSONDecoder()

# b= jd.decode(h[-1]['details'])
# get_card_info(b['rounds'][0]['actions'][0]['initiator'])

# dpt_r_corr(h)
# plt_rating(h)
# plt_dpt(h)





# [{'uid': 'starter-388-fkxda', 
# 	'xp': 1, 'gold': False, 
# 	'card_detail_id': 388, 
# 	'level': 1, 
# 	'edition': 7, 
# 	'state': {'alive': True, 'stats': [2, 0, 0, 6, 7, 4], 
# 	'base_health': 7, 'other': []}, 'abilities': ['Trample']}, 
# {'uid': 'starter-172-rHPA9', 
# 	'xp': 1, 
# 	'gold': False, 
# 	'card_detail_id': 172, 
# 	'level': 1, 
# 	'edition': 4, 
# 	'state': {'alive': True, 'stats': [0, 0, 1, 0, 1, 3], 
# 	'base_health': 1, 
# 	'other': []}, 'abilities': ['Flying']}, 
# {'uid': 'starter-382-1HgdI', 'xp': 1, 'gold': False, 'card_detail_id': 382, 'level': 1, 'edition': 7, 'state': {'alive': True, 'stats': [0, 2, 0, 0, 4, 2], 'base_health': 4, 'other': []}, 'abilities': []}]




































#