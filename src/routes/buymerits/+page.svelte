<script lang="ts">
	import * as Table from '@/components/ui/table';
	import { AMRAuction, MeritPurchase, Rocket } from '@/event_helpers/rockets';
	import { ndk } from '@/ndk';
	import { bitcoinTip, getIncomingTransactions, txs } from '@/stores/bitcoin';
	import { currentUser } from '@/stores/session';
	import { NDKEvent } from '@nostr-dev-kit/ndk';
	import { Avatar } from '@nostr-dev-kit/ndk-svelte-components';
	import { onDestroy } from 'svelte';
	import { derived } from 'svelte/store';
	import AssociateBitcoinAddress from '../../components/AssociateBitcoinAddress.svelte';
	import Heading from '../../components/Heading.svelte';
	import Login from '../../components/Login.svelte';
	import MeritAuctions from '../../stateupdaters/MeritAuctions.svelte';
	import BuyAmr from '../../components/BuyAMR.svelte';
	import { goto } from '$app/navigation';
	import { base } from '$app/paths';
	import * as Card from '@/components/ui/card';
	import { Badge } from '@/components/ui/badge';
	let rocketEvents = $ndk.storeSubscribe([{ kinds: [31108 as number] }], { subId: 'all_rockets' });
	onDestroy(() => {
		rocketEvents?.unsubscribe();
	});

	let dev = false;

	let rockets = derived(rocketEvents, ($rocketEvents) => {
		let m = new Map<string, NDKEvent>();
		for (let e of $rocketEvents) {
			let existing = m.get(e.pubkey + e.dTag);
			if (!existing) {
				existing = e;
			}
			if (e.created_at > existing) {
				existing = e;
			}
			m.set(e.pubkey + e.dTag, e);
		}
		return Array.from(m, ([_, e]) => new Rocket(e));
	});

	let pendingSales = derived(rockets, ($rockets) => {
		let merits = new Map<Rocket, AMRAuction[]>();
		for (let r of $rockets) {
			let _auctions: AMRAuction[] = r.PendingAMRAuctions();
			merits.set(r, _auctions);
		}

		return merits;
	});

	let _transactions = new Map<string, txs>();
	let transactions = derived(pendingSales, ($pendingSales) => {
		for (let [r, s] of $pendingSales) {
			for (let amr of s) {
				if (!_transactions.get(amr.RxAddress)) {
					_transactions.set(amr.RxAddress, new txs(amr.RxAddress));
				}
				let existing = _transactions.get(amr.RxAddress)!;
				if (Math.floor(new Date().getTime() / 1000) > existing.LastAttempt + 10000) {
					existing.LastAttempt = Math.floor(new Date().getTime() / 1000);
					getIncomingTransactions(amr.RxAddress)
						.then((result) => {
							if (result) {
								existing.LastUpdate = Math.floor(new Date().getTime() / 1000);
							}
							if (result.length > 0) {
								existing.Data = result;
								_transactions.set(amr.RxAddress, existing);
								_transactions = _transactions;
							}
						})
						.catch((c) => {
							console.log(c);
						});
				}
			}
		}
		return _transactions;
	});

	let nextSoldButNotInState = derived(
		[pendingSales, transactions, bitcoinTip, currentUser],
		([$pendingSales, $transactions, $bitcoinTip, $currentUser]) => {
			if ($currentUser) {
				for (let [r, p] of $pendingSales) {
					if (r.VotePowerForPubkey($currentUser.pubkey) > 0) {
						for (let amrAuction of p) {
							if (
								amrAuction.Status(r, $bitcoinTip.height, $transactions.get(amrAuction.RxAddress)) ==
								'SOLD & PENDING RATIFICATION'
							) {
								let txs = $transactions.get(amrAuction.RxAddress);
								if (txs) {
									for (let [address, txo] of txs.From()) {
										for (let [_, ba] of r.BitcoinAssociations()) {
											if (ba.Address == txo.From) {
												return new MeritPurchase(r, amrAuction, ba.Pubkey, txo);
											}
										}
									}
								}
							}
						}
					}
				}
			}
		}
	);

	nextSoldButNotInState.subscribe((t) => {
		if (t) {
			//console.log(t.rocket.UpsertMeritTransfer(t)?.rawEvent());
			let e = t.rocket.UpsertMeritTransfer(t);
			if (e) {
				e.publish().then((x) => {
					console.log(goto(`${base}/${new Rocket(e).URL()}`));
				});
			}
			//t.rocket.UpsertMeritTransfer(t)?.publish()
		}
	});

	let nostrocket = derived(rockets, ($rockets) => {
		let rocket: Rocket | undefined = undefined;
		for (let r of $rockets) {
			if (
				r.Name() == 'NOSTROCKET' &&
				r.Event.pubkey == 'd91191e30e00444b942c0e82cad470b32af171764c2275bee0bd99377efd4075'
			) {
				//we consume the current list of bitcoin addresses from Nostrocket as a service so that users don't need to add a new address for every rocket
				//todo: make this dependent on votepower not my pubkey
				//todo: also allow rockets to have their own list of addresses so they can be used without nostrocket
				rocket = r;
			}
		}
		return rocket;
	});

	transactions.subscribe((t) => {});

	function getRocketClass(rocket: Rocket): string {
		return rocket.Testnet() ? 'dark:border-red-600' : '';
	}
</script>

{#if $nostrocket}<AssociateBitcoinAddress rocket={$nostrocket} />
{/if}

{#if $currentUser}
	{#each $pendingSales as [rocket, amr] (rocket.Event.id)}
		{#if amr.length > 0}
			<Card.Root class={getRocketClass(rocket)}>
				<Card.CardHeader>
					<div class="flex flex-nowrap place-items-stretch">
						<h3 class=" mr-auto text-nowrap text-2xl">
							{`ROCKET: ${rocket.Name().toUpperCase()}`}
						</h3>
						{#if rocket.Testnet()}<Badge
								variant="destructive"
								class="flex h-8 shrink-0 items-center justify-center rounded-sm"
							>
								<span
									on:click={() => {
										dev = true;
										alert(
											'dev mode enabled, refresh page if this was unintentional or you may lose sats'
										);
									}}>TESTNET</span
								>
							</Badge>{/if}
					</div>
					<Heading></Heading>
					<Table.Root>
						<Table.Header>
							<Table.Row>
								<Table.Head class="w-[10px]">Seller</Table.Head>
								<Table.Head class="w-[10px]">AMR</Table.Head>
								<Table.Head class="w-[10px]">Merits</Table.Head>
								<Table.Head class="w-[150px] text-right">Current Price (sats)</Table.Head>
								<Table.Head>Status</Table.Head>
								<Table.Head>Receiving Address</Table.Head>
								<Table.Head></Table.Head>
							</Table.Row>
						</Table.Header>
						<Table.Body>
							{#each amr as p (p.AMRIDs)}
								<Table.Row>
									<Table.Cell
										><Avatar
											ndk={$ndk}
											pubkey={p.Owner}
											class="aspect-square w-10 flex-none rounded-full object-cover"
										/></Table.Cell
									>
									<Table.Cell
										>{p.AMRIDs.length > 1 ? 'multiple' : p.AMRIDs[0].substring(0, 12)}</Table.Cell
									>
									<Table.Cell>{p.Merits}</Table.Cell>
									<Table.Cell class="text-right">{p.Merits}</Table.Cell>
									<Table.Cell
										>{p.Status(
											rocket,
											$bitcoinTip.height,
											$transactions.get(p.RxAddress)
										)}</Table.Cell
									>
									<Table.Cell
										on:click={() => {
											console.log($transactions.get(p.RxAddress)?.From());
										}}>{p.RxAddress}</Table.Cell
									>
									<Table.Cell
										>{#if p.Status(rocket, $bitcoinTip.height, $transactions.get(p.RxAddress)) == 'OPEN' && (!rocket.Testnet() || dev)}<BuyAmr
												auction={p}
											/>{/if}</Table.Cell
									>
								</Table.Row>
							{/each}
						</Table.Body>
					</Table.Root>
				</Card.CardHeader>
			</Card.Root>
		{/if}
	{/each}
{:else}<Login />{/if}
<MeritAuctions {rockets} />
