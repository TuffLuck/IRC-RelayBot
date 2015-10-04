#!/usr/bin/python
# coding=UTF-8

import os
import socket
import ssl
import random
import re
from errno import EAGAIN
from math import ceil
from sys import exc_info
from time import sleep, time

cc = {
	'B' : 0x02,
	'C' : {
		'White' :   '\x0300', 'white' :   '\x0315',
		'Black' :   '\x0301', 'black' :   '\x0314',
		'Magenta' : '\x0306', 'magenta' : '\x0313',
		'Red' :     '\x0305', 'red' :     '\x0304',
		'Yellow' :  '\x0307', 'yellow' :  '\x0308',
		'Green' :   '\x0303', 'green' :   '\x0309',
		'Cyan' :    '\x0310', 'cyan' :    '\x0311',
		'Blue' :    '\x0302', 'blue' :    '\x0312',
		'normal' :  '\x03'
	},
	'O' : 0x0f,
	'R' : 0x16,
	'U' : 0x1f,
}

class Relay ():
	def __init__ (self, from_server, to_server,
	 from_channel, to_channel):
		self.from_server = from_server
		self.to_server = to_server
		self.from_channel = from_channel
		self.to_channel = to_channel
		self.message = lambda x: self.from_server.message(
			self.to_server.name, from_channel, x)
		self.from_server.raw_line("JOIN %s" % from_channel)
		self.to_server.raw_line("JOIN %s" % to_channel)

class Home ():

	def __init__ (self):
		self.trusted=[
		 r".*!\^tuffluck@.*",
		 r".*!x@freal\.xyz",
		 r".*!x@gonullyourself.com",
		]
		self.relayfile="relays.db"

		self.remotes={}
		self.idle=time()

		try:
			with open(self.relayfile, "r") as f:
				for line in f.readlines(): self.read_conf(line.split())
		except IOError:
			self.remotes["IRCNETWORKHERE"] = Remote(
				self,
				name = "IRC-NETWORK-NAME",
				server = ("127.0.0.1", 9999),
				nick = ">",
				ident = "TLRelayBot",
				throttle = 0,
				identify_command = "SQUERY COMMAND",
				ssl_enabled = True
			)

	def read_conf (self, line, rehash = False):
		if not line: return
		if not line[0]: return
		if line[0] == "S" and len(line) >= 7:
			if rehash and line[1] in self.remotes:
				self.remotes[line[1]].throttle = float(line[6])
			else:
				self.remotes[line[1]] = Remote(
					parent = self,
					name = line[1],
					server = (line[2], int(line[3][line[3][0] == "+":])),
					nick = line[4],
					ident = line[5],
					throttle = float(line[6]),
					identify_command = " ".join(line[7:]) if len(line) > 7 else None,
					ssl_enabled = line[3][0] == "+"
				)
		elif line[0] == "R" and len(line) == 5:
			if rehash: pass
			else:
				if not line[4] in self.remotes[line[2]].relays:
					self.remotes[line[2]].relays[line[4]] = {}
				self.remotes[line[2]].relayqueue.append(tuple(line[1:]))

	def write_conf (self):
		print "writing conf"
		with open(self.relayfile, "w") as f:
			for name, remote in self.remotes.iteritems():
				f.write("S %s %s %s%s %s %s %f%s\n"
				 % (name,
				 remote.server[0],
				 "+" if remote.ssl_enabled else "",
				 remote.server[1],
				 remote.nick,
				 remote.ident,
				 remote.throttle,
				 " "+remote.identify_command if remote.identify_command else ""
				))
			for remote in self.remotes.itervalues():
				for relays in remote.relays.itervalues():
					# sorry for the nesting
					for relay in relays.itervalues():
						f.write("R %s %s %s %s\n"
						 % (relay.from_server.name, relay.to_server.name,
						 relay.from_channel, relay.to_channel)
						)

	def loop (self):
		now = time()
		if self.idle + 1 < now: sleep(1)
		for r in self.remotes.values():
#				try:
			if r.reconnect_in == None: pass
			elif now > r.reconnect_in: r.reconnect()

			if r.connected: r.loop()
#				except BaseException:
#					type,e,tb=exc_info()
#					self.buzz(W+"%s\x0F %s in <%s:%d> %s" % (w.nick, type,
#					 tb.tb_frame.f_code.co_filename, tb.tb_lineno, e.message))
#					self.buzz(E+"%s\x0F killed" % w.nick)
#					w.connected=False


	def run (self):
		try:
			while 1: self.loop()
		except KeyboardInterrupt:
			self.write_conf()

		for r in self.remotes.itervalues():
			r.raw_line("QUIT")

		while self.remotes:
			for i, r in self.remotes.items():
				try:
					if r.connected: r.loop()
					else: del self.remotes[i]
				except socket.error as e:
					r.connected=False

class Remote ():
	def __init__ (self, parent, name, server, nick, ident, throttle,
	 identify_command, ssl_enabled = False):
		now = time()
		self.parent = parent
		self.name = name
		self.server = server
		self.nick = nick
		self.ident = ident
		self.throttle = throttle
		self.identify_command = identify_command
		self.relays = {}
		self.members = {}
		self.flags = {}
		self.mutes = {}

		self.sock=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
		self.ssl_enabled=ssl_enabled
		if ssl_enabled:
			self.sock = ssl.wrap_socket(self.sock)

		self.buffer=""
		self.sendqueue=[]
		self.relayqueue=[]
		self.connected=False
		self.registered=False
		self.start = now
		self.nodata = 0
		self.last = now
		
		self.reconnect_in = 0
		print "created remote"

	def reconnect (self):
		try:
			print "connecting to %s:%d" % self.server
			self.sock.settimeout(10)
			self.sock.connect(self.server)
			self.raw_line("USER %s 0 0 :%s" % ((self.ident,) * 2))
			self.raw_line("NICK %s" % self.nick)
			self.connected=True
			self.sock.setblocking(0)
		except socket.error as e:
			self.connected = False
		except ValueError:
			print "couldn't reconnect to %s:%d" % self.server
			self.connected = True

		self.reconnect_in = None

	def raw_line (self, line):
		try:
			return self.sock.send(
				line[:300] +
				(cc['C']['yellow']+"[CUT]" if len(line)>300 else "") +
				("\x01" if line[-1] == "\x01" else "") +
				"\r\n"
			)
		except ssl.SSLWantWriteError:
			return
		except socket.error as e:
			if e.errno == EAGAIN: return
			print str(e)
			self.connected=False

	def message (self, server, channel, line):
		if channel not in self.flags: return
		if "m" in self.flags[channel]: return
		self.sendqueue.append((server, channel, line))

	def parse_command (self, a, t, m, p):
		if not m: return
		if m[0] != ">": return
		m = m[1:]
		t = t.lower()
		

#		if m == "names":
#			if t not in self.members: return
#			for relay in self.relays[t].itervalues():
#				names = ""
#				for name in relay.from_server.members[t]
#				self.raw_line("PRIVMSG %s :
#		elif not a: return
		if not a: return
		if m == "addserver" and p and len(p) > 4:
			spl_port = p[1].find("/")
			if spl_port == -1:
				self.raw_line("PRIVMSG %s :server must be in format host/port"
				 % t)
				return
			is_ssl = p[1][spl_port+1] == "+"
			try: float(p[4])
			except TypeError:
				self.raw_line("PRIVMSG %s :msg throttle must be a float or int"
				 % t)
				return
			self.raw_line("PRIVMSG %s :added server. connecting...." % t)
			self.parent.remotes[p[0]] = Remote(
				parent = self.parent,
				name = p[0],
				server = (p[1][:spl_port], int(p[1][spl_port + 1 + is_ssl:])),
				nick = p[2],
				ident = p[3],
				throttle = float(p[4]),
				identify_command = p[5:] if len(p) > 5 else None,
				ssl_enabled = is_ssl
			)
		elif m == "addserver":
			self.raw_line("PRIVMSG %s :it is %c>addserver %cname%c "
			 "%cserver/[+]port%c %cnick%c %cident%c %cthrottle%c "
			 "%c[identify-command]"
			 % ((t, cc['B']) + (cc['U'],) * 11))
		elif m == "addrelay" and p and len(p) >= 1:
			if p[0] not in self.parent.remotes:
				self.raw_line("PRIVMSG %s :%c%s%c not in servers (server "
				 "names are case sensitive)" % (t, cc['B'], p[0], cc['O']))
				return
#			if ( (p[0], t) in self.relays and
#			 t2 in self.relays[(p[0], t)].channels and
#			 self.name in self.relays[(p[0], t)].servers:
#				self.raw_line
			self.raw_line("PRIVMSG %s :relay added. joining and relaying "
			 "from there to here...." % t)
			t2 = p[1].lower() if len(p) == 2 else t
			if not t2 in self.parent.remotes[p[0]].relays:
				self.parent.remotes[p[0]].relays[t2] = {}
			self.parent.remotes[p[0]].relays[t2][(self.name, t)] = Relay(
				from_server = self,
				to_server = self.parent.remotes[p[0]],
				from_channel = t,
				to_channel = t2)
		elif m == "addrelay":
			self.raw_line("PRIVMSG %s :it is %c>addrelay %cremote server%c "
			 "%c[remote channel]"
			 % ((t, cc['B']) + (cc['U'],) * 3))
		elif m == "connect" and p:
			if p[0] not in self.parent.remotes:
				self.raw_line("PRIVMSG %s :%c%s%c not in servers (server "
				 "names are case sensitive)" % (t, cc['B'], p[0], cc['O']))
				return
			self.parent.remotes[p[0]].reconnect_in = time() + 10
		elif m == "disconnect":
			self.raw_line("QUIT")
			self.sock.close()
			self.sock=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
			self.ssl_enabled=ssl_enabled
			if ssl_enabled:
				self.sock = ssl.wrap_socket(self.sock)
		elif m == "die":
			raise KeyboardInterrupt
		elif m == "lsserver":
			for remote in self.parent.remotes.itervalues():
				self.raw_line("PRIVMSG %s :%s: %s!%s@%s:%s%d"
				 % (t, remote.name, remote.nick, remote.ident, remote.server[0],
				 "+" if remote.ssl_enabled else "", remote.server[1]))
			self.raw_line("PRIVMSG %s :connected to %d servers" %
			 (t, len(self.parent.remotes)))
		elif m == "lsrelay":
			i = 0
			if t not in self.relays: return
			for relay in self.relays[t].itervalues():
				self.raw_line("PRIVMSG %s :to %s:%s"
				 % (t, relay.from_server.name, relay.from_channel))
				i += 1
			if not i:
				self.raw_line("PRIVMSG %s :this channel is not being relayed out"
				 % t)
		elif m == "mute" and p:
			mask = p[0].lower()
			if t not in self.mutes: return
			self.mutes[t].add(mask)
			self.raw_line("PRIVMSG %s :mask /%s/ muted" % (t, mask))
		elif m == "mutes":
			if t not in self.mutes: return
			for i in self.mutes[t]:
				self.raw_line("PRIVMSG %s :mask /%s/ muted" % (t, i))
		elif m == "unmute" and p:
			mask = p[0].lower()
			if t not in self.mutes: return
			if mask not in self.mutes[t]:
				self.raw_line("PRIVMSG %s :could not find mask %s" % (t, mask))
			self.mutes[t].remove(mask)
			self.raw_line("PRIVMSG %s :mask /%s/ unmuted" % (t, mask))
		elif m == "part":
			self.raw_line("PART %s" % t)
		elif m == "rehash":
			try:
				with open(self.relayfile, "r") as f:
					for line in f.readlines(): self.read_conf(line.split())
				self.raw_line("PRIVMSG %s :rehashed" % t)
			except IOError as e:
				self.raw_line("PRIVMSG %s :could not write to file: %s"
				 % (t, e.message))
		elif m == "rmserver" and p:
			if p[0] not in self.parent.remotes:
				self.raw_line("PRIVMSG %s :server %c%s%c not found (names are "
				 "case sensitive)" % (t, cc['B'], p[0], cc['O']))
				return
			self.raw_line("PRIVMSG %s :removed server. disconnecting...." % t)
			self.parent.remotes[p[0]].raw_line("QUIT")
			del self.parent.remotes[p[0]]
		elif m == "rmserver":
			self.raw_line("PRIVMSG %s :it is %c>rmserver %cname"
			 % (t, cc['B'], cc['U']))
		elif m == "rmrelay" and p and len(p) >= 2:
			t2 = p[1].lower()
			if p[0] not in self.parent.remotes:
				self.raw_line("PRIVMSG %s :server not found (case sensitive)!"
				 % t)
				return
			if (t2 not in self.parent.remotes[p[0]].relays or
			 (self.name, t) not in self.parent.remotes[p[0]].relays[t2]):
				self.raw_line("PRIVMSG %s :i can't find a relay to here!" % t)
				return
			self.raw_line("PRIVMSG %s :removed relay" % t)
			del self.parent.remotes[p[0]].relays[t2][(self.name, t)]
		elif m == "save":
			self.parent.write_conf()
			self.raw_line("PRIVMSG %s :config written" % t)
		elif m == "trust" and p:
			self.parent.trusted.append(p[0])
			self.raw_line("PRIVMSG %s :trusted mask %s" % (t, p[0]))
		elif m == "untrust" and p:
			if p[0] not in self.parent.trusted:
				self.raw_line("PRIVMSG %s :%s was not trusted!" % (t, p[0]))
				return
			self.parent.trusted.remove(p[0])
			self.raw_line("PRIVMSG %s :untrusted mask %s" % (t, p[0]))
#		elif m == "set" and p:
#			try: int(p[0])
#			except TypeError:
#				self.raw_line("PRIVMSG %s :relay ID must be a number" % t)
#				return
#			if len(p) == 1 and i < len(self.parent.relays):
#				relay = self.parent.relays[i]
#				self.raw_line("PRIVMSG %s :relay from %s:%s to %s:%s has the "
#				 "following settings:"
#				 % (t, relay.servers[0], relay.channels[0],
#				 relay.servers[1], relay.channels[1]))
#				self.raw_line("PRIVMSG %s :%srelaying to %s, %srelaying to %s"
#				 % (t, "" if relay.sendto[0] else "NOT ", relay.channels[0],
#				 "" if relay.sendto[1] else "NOT ", relay.channels[1]))

	def send_related (self, channel):
		if channel not in self.relays: return
		for c in self.relays[channel].itervalues():
			yield c.message

	def handle_line (self, line):
		now = time()
		self.parent.idle = now
		line = line.split(" ")
		command = line[0].upper()

		if command == "ERROR":
			self.reconnect_in = now + 10

		elif command == "PING":
			self.raw_line("PONG %s" % line[1])

		elif len(line) > 1:
			command = line[1].upper()
			if command == "001":
				sleep(1)
				self.raw_line("MODE %s +B-Ciw" % self.nick)
				if self.identify_command: self.raw_line(self.identify_command)

				for relay in self.relayqueue:
					self.relays[relay[3]][(relay[0], relay[2])] = Relay(
						from_server = self.parent.remotes[relay[0]],
						to_server = self.parent.remotes[relay[1]],
						from_channel = relay[2],
						to_channel = relay[3]
					)
			
#			elif re.search("^:[^ ]+ 376 ", " ".join(line)):
#				self.raw_line("JOIN %s %s" % (self.hive[0], self.hive[1]))

			elif command in ("401", "404"):
				c=line[3].lower()
				self.flags[c] += "m"

			# erroneous nickname
			elif command == "432":
				self.connected=False

			elif command == "433":
				self.nick += "_"
				self.raw_line("NICK %s" % self.nick)

			elif command == "482":
				c = line[3].lower()
				self.flags[c] += "t"

			elif command == "NICK":
				Oldnick = line[0].split("!")[0].lstrip(":")
				oldnick = Oldnick.lower()
				newnick = line[2].lstrip(":")
				if oldnick == self.nick.lower():
					self.nick = newnick
				for channel, members in self.members.iteritems():
					if oldnick not in members: continue
					members.remove(oldnick)
					members.append(newnick.lower())
					for send in self.send_related(channel):
						send("%s( Nick change ) [ %s -> %s ]"
						 % (cc['C']['Cyan'], Oldnick, newnick))

			elif command == "JOIN":
				nuh=line[0].lstrip(":").split("!")
				n=nuh[0].lower()
				c=line[2].lstrip(":").lower()
				if n==self.nick.lower():
					self.members[c] = []
					self.flags[c] = ""
					self.mutes[c] = set()
				else:
					for send in self.send_related(c):
						send("%s( Joins ) [ %c%s%c!%s ]"
						 % (cc['C']['Green'], cc['B'], nuh[0], cc['B'], nuh[1]))
				if n not in self.members[c]:
					self.members[c].append(n)

			elif command in ("PART","KICK"):
				N = line[0].lstrip(":").split("!")[0]
				n = N.lower()
				c = line[2].lower()
				kick = command == "KICK"
				m = " ".join(line[3 + kick:])[1:]
				if c not in self.members: return
				if n in self.members[c]: self.members[c].remove(n)
				if n==self.nick.lower():
					del self.members[c]
					del self.flags[c]
					del self.mutes[c]
				for send in self.send_related(c):
					send("%s( %s ) [ %c%s%c%s ]"
					 % (cc['C']['Yellow'],
					 "Kicks by %s" % N if kick else "Parts",
					 cc['B'], line[3] if kick else N, cc['B'],
					 " for %c%s%c%s" % (cc['B'], m, cc['O'],
					 cc['C']['Yellow']) if m else ""))

			elif command=="MODE":
				N=line[0].lstrip(":").split("!")[0]
				c=line[2].lower()
				if c not in self.relays: return
				self.flags[c]=""
				for send in self.send_related(c):
					send("%s( Modes by %s ) [ %s ]"
					 % (cc['C']['Magenta'], N, " ".join(line[3:]).rstrip()))

			elif command=="QUIT":
				N = line[0].lstrip(":").split("!")[0]
				n = N.lower()
				m = " ".join(line[2:])[1:]
				for c in self.members.keys():
					if n not in self.members[c]: continue
					self.members[c].remove(n)
					for send in self.send_related(c):
						send("%s( Quits ) [ %c%s%c%s ]"
						 % (cc['C']['Red'], cc['B'], N, cc['B'],
						 " for %c%s%c%s" % (cc['B'], m, cc['O'],
						 cc['C']['Red']) if m else ""))

			elif command == "INVITE":
				a = False
				for i in self.parent.trusted:
					if re.match("^:"+i+"$",line[0]):
						a = True
						break
				if not a: return
				self.raw_line("JOIN %s" % line[3])

			elif command == "PRIVMSG":
				c = line[2].lower()
				N = line[0].lstrip(":").split("!")[0]
				n = N.lower()
				m = " ".join(line[3:])[1:]
				a = False
				fullhost = line[0].lstrip(":").lower()
				for i in self.parent.trusted:
					if re.match("^"+i+"$", fullhost):
						a = True
						break
				self.parse_command(
				 a,
				 line[0].split("!")[0].lstrip(":") if c == self.nick.lower(
				 ) else c,
				 line[3].lstrip(":").lower(),
				 line[4:] if len(line)>4 else None)
				if c not in self.relays: return
				for mute in self.mutes[c]:
					if re.search(mute, fullhost): return
				is_ctcp = bool(m and m[0] == "\x01")
				is_action = is_ctcp and not m.find("\x01ACTION ")
				prefix = (
					"<%c%s%c>",
					"CTCP(%c%s%c)",
					"* %c%s%c"
				)[
					is_ctcp + is_action
				] % (cc['B'], N, cc['O'])

				if is_ctcp: m = m[1:-1]
				if is_action: m = m[7:]

				for send in self.send_related(c):
					send("%s %s" % (prefix, m))

			elif command == "NOTICE":
				c = line[2].lower()
				if c not in self.relays: return
				fullhost = line[0].lstrip(":").lower()
				for mute in self.mutes[c]:
					if re.search(mute, fullhost): return
				N = line[0].lstrip(":").split("!")[0]
				n = N.lower()
				m = " ".join(line[3:])[1:]
				is_nctcp = bool(m and m[0] == "\x01")
				prefix = (
					"![%c%s%c]",
					"NCTCP[%c%s%c]"
				)[
					is_nctcp
				] % (cc['B'], n, cc['O'])
				if is_nctcp: m = m[1:-1]
				for send in self.send_related(c):
					send("%s %s" % (prefix, m))

			elif line[1] in ("002", "003", "004",
			 "251", "252", "254", "255", "265", "266",
			 "372", "375", "376"):
				pass

			else:
				print "unhandled: %s" % line

	def check_sendqueue (self):
		now = time()
		for item in self.sendqueue:
			if self.last + self.throttle > now: continue
			self.raw_line("PRIVMSG %s :\x01ACTION %s%s%c:%s\x01" %
			 (item[1], cc['C']['black'], item[0], cc['O'], item[2]))
			self.sendqueue.pop(0)
			self.last = now
			break

	def loop (self):
		try:
			self.buffer += self.sock.recv(512)
			data=self.buffer.split("\r\n")
			self.buffer=data.pop()
			if not data:
				self.nodata += 1
				if self.nodata > 3:
					self.connected=False
			else:
				self.nodata=0
			for line in data:
				self.handle_line(line)
		except ssl.SSLWantReadError:
			self.check_sendqueue()
		except ssl.SSLWantWriteError:
			pass
		except socket.error as e:
			if e.errno == EAGAIN:
				self.check_sendqueue()
			print "socket error: %s" % e.message
			if not self.registered and self.start + 10 < time():
				self.connected=False

relay=Home().run()
