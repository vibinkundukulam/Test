# Scraper for Amazon

# made in 30 min

# This code can be considered part of the public domain, for any use by anyone.


import requests, re, sys
from bs4 import BeautifulSoup
from mongoengine import *

connect('amazon')
# connect('amazon', host='mongodb://pennapps:pennapps1@ds037617-a.mongolab.com:37617/data_scraping')

class amazon_item(Document):
  meta = {'collection': 'amazon'}
	title = StringField()
	price = StringField()
	save_price = StringField()
	save_percent = StringField()
	url = URLField()


def url(query):
	return r'http://www.amazon.com/s/ref=nb_sb_noss_1/185-7608369-3271665?url=search-alias%3Daps&field-keywords=' + str(query)


def get_all_item_urls(query):

	r = requests.get(url(query), config={'verbose': sys.stderr})
	soup = BeautifulSoup(r.text, 'html5lib')

	# get all rows
	thelist = soup.find_all('div', 'rslt', id=re.compile(r'result_\d+?'))
	thelist.append(soup.find('div', 'fstRow'))

	urls = []
	for item in thelist:
		#print item

		if 'href' in item.h3.a.attrs and item.h3.a.attrs['href'][:3] != r'/b/':
			urls.append(item.h3.a.attrs['href'])

	return urls

def item_info_from_url(url):

	print url
	r = requests.get(url,config={'verbose': sys.stderr})
	soup = BeautifulSoup(r.text)

	info = {}

	info['title'] = soup.find(id='btAsinTitle').text.strip()
	info['price'] = soup.find(id='actualPriceValue').b.text.strip()
	info['save_price'] = re.search(r'\$\d*?\.\d{2}',soup.find(id='youSaveValue').text).group(0)
	info['save_percent'] = re.search(r'(?<=\()\d+\%(?=\))', soup.find(id='youSaveValue').text).group(0)
	info['url'] = url

	return info

def save_item(info):

	item = amazon_item(**info)
	item.save()


def main():
	query = 'ipod'

	urls = get_all_item_urls(query)
	for url in urls:
		try:
			info = item_info_from_url(url)
			print info
			save_item(info)
		except AttributeError:
			continue

if __name__ == '__main__':
	main()
