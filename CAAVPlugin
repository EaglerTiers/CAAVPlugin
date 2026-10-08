package com.caav;

import java.io.File;
import java.io.IOException;
import java.util.*;
import org.bukkit.*;
import org.bukkit.block.Block;
import org.bukkit.command.*;
import org.bukkit.configuration.ConfigurationSection;
import org.bukkit.configuration.file.YamlConfiguration;
import org.bukkit.entity.Entity;
import org.bukkit.entity.Player;
import org.bukkit.entity.Villager;
import org.bukkit.event.*;
import org.bukkit.event.block.*;
import org.bukkit.event.entity.EntityDamageEvent;
import org.bukkit.event.inventory.*;
import org.bukkit.event.player.*;
import org.bukkit.inventory.*;
import org.bukkit.inventory.meta.ItemMeta;
import org.bukkit.plugin.java.JavaPlugin;

public class CAAVPlugin extends JavaPlugin implements Listener, CommandExecutor {
    private File dataFile;
    private YamlConfiguration data;
    private final Map<UUID, List<Location>> selections = new HashMap<UUID, List<Location>>();
    private final Map<UUID, String> editingVillager = new HashMap<UUID, String>();
    private final Map<UUID, String> tradeAction = new HashMap<UUID, String>();
    private final Map<UUID, Integer> selectedTrade = new HashMap<UUID, Integer>();
    private final Map<UUID, UUID> villagerIds = new HashMap<UUID, UUID>();
    private final Map<UUID, String> villagerNames = new HashMap<UUID, String>();
    private final String PREFIX = ChatColor.GOLD + "[CAAV] " + ChatColor.GRAY;

    @Override public void onEnable() {
        if (!getDataFolder().exists()) getDataFolder().mkdirs();
        dataFile = new File(getDataFolder(), "data.yml");
        data = YamlConfiguration.loadConfiguration(dataFile);
        getServer().getPluginManager().registerEvents(this, this);
        getCommand("caav").setExecutor(this);
        getCommand("area").setExecutor(this);
        getCommand("vnpc").setExecutor(this);
        loadVillagerIndex();
        getLogger().info("CAAVPlugin enabled.");
    }
    @Override public void onDisable() { saveData(); }

    private void loadVillagerIndex() {
        ConfigurationSection s = data.getConfigurationSection("villagers");
        if (s == null) return;
        for (String key : s.getKeys(false)) {
            try {
                UUID id = UUID.fromString(key);
                String name = data.getString("villagers." + key + ".name", key);
                villagerIds.put(id, id); villagerNames.put(id, name.toLowerCase(Locale.ROOT));
            } catch (Exception ignored) {}
        }
    }
    private void saveData() {
        try { data.save(dataFile); } catch (IOException e) { getLogger().severe("Could not save data.yml: " + e.getMessage()); }
    }
    private boolean admin(CommandSender s) {
        if (s.hasPermission("caav.admin")) return true;
        s.sendMessage(PREFIX + ChatColor.RED + "You need permission caav.admin.");
        return false;
    }
    private ItemStack named(Material m, String name, String... lore) {
        ItemStack it = new ItemStack(m); ItemMeta meta = it.getItemMeta();
        meta.setDisplayName(ChatColor.YELLOW + name);
        if (lore != null && lore.length > 0) meta.setLore(Arrays.asList(lore));
        it.setItemMeta(meta); return it;
    }
    private boolean isNamed(ItemStack it, String name) {
        return it != null && it.hasItemMeta() && it.getItemMeta().hasDisplayName()
            && ChatColor.stripColor(it.getItemMeta().getDisplayName()).equalsIgnoreCase(name);
    }
    private String path(UUID id) { return "villagers." + id.toString(); }

    @Override public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        String cmd = command.getName().toLowerCase(Locale.ROOT);
        if (cmd.equals("caav")) {
            sender.sendMessage(ChatColor.GOLD + "----- CAAV Commands -----");
            sender.sendMessage(ChatColor.YELLOW + "/area edit" + ChatColor.GRAY + " - get axe and select four corners");
            sender.sendMessage(ChatColor.YELLOW + "/vnpc create <name>" + ChatColor.GRAY + " - spawn custom villager");
            sender.sendMessage(ChatColor.YELLOW + "/vnpc edit <name>" + ChatColor.GRAY + " - edit trades in GUI");
            sender.sendMessage(ChatColor.YELLOW + "/vnpc claim <name>" + ChatColor.GRAY + " - collect all paid items");
            return true;
        }
        if (!sender.hasPermission("caav.use")) { sender.sendMessage(PREFIX + ChatColor.RED + "You do not have permission."); return true; }
        if (cmd.equals("area")) {
            if (!admin(sender)) return true;
            if (!(sender instanceof Player)) { sender.sendMessage("Players only."); return true; }
            if (args.length == 1 && args[0].equalsIgnoreCase("edit")) {
                Player p = (Player)sender;
                ItemStack axe = named(Material.WOOD_AXE, "CAAV Area Selector", "Right-click four corner blocks", "Area protection is full-height.");
                p.getInventory().addItem(axe);
                selections.put(p.getUniqueId(), new ArrayList<Location>());
                p.sendMessage(PREFIX + "Right-click the four corner blocks of your area in order.");
                p.sendMessage(PREFIX + "Click the fourth corner to save the protected polygon.");
            } else sender.sendMessage(PREFIX + "Usage: /area edit");
            return true;
        }
        if (cmd.equals("vnpc")) {
            if (!admin(sender)) return true;
            if (args.length < 2) { sender.sendMessage(PREFIX + "Usage: /vnpc create <name> | edit <name> | claim <name>"); return true; }
            String action = args[0].toLowerCase(Locale.ROOT);
            String name = join(args, 1);
            if (action.equals("create")) {
                if (!(sender instanceof Player)) { sender.sendMessage("Players only."); return true; }
                Player p = (Player)sender;
                Villager v = p.getWorld().spawn(p.getLocation(), Villager.class);
                v.setCustomName(ChatColor.GREEN + name); v.setCustomNameVisible(true);
                v.setRemoveWhenFarAway(false);
                String key = path(v.getUniqueId());
                data.set(key + ".name", name);
                data.set(key + ".world", v.getWorld().getName());
                data.set(key + ".trades", new ArrayList<String>());
                data.set(key + ".stock", new ArrayList<String>());
                villagerIds.put(v.getUniqueId(), v.getUniqueId());
                villagerNames.put(v.getUniqueId(), name.toLowerCase(Locale.ROOT));
                saveData();
                sender.sendMessage(PREFIX + "Created villager '" + name + "'. Use /vnpc edit " + name);
                return true;
            }
            UUID id = findVillager(name);
            if (id == null) { sender.sendMessage(PREFIX + ChatColor.RED + "No saved villager named '" + name + "'."); return true; }
            if (action.equals("edit")) {
                if (!(sender instanceof Player)) return true;
                Player p = (Player)sender;
                editingVillager.put(p.getUniqueId(), id.toString());
                openEditor(p, id);
            } else if (action.equals("claim")) {
                if (!(sender instanceof Player)) return true;
                Player p = (Player)sender;
                List<?> stock = data.getList(path(id) + ".stock", new ArrayList<Object>());
                int claimed = 0;
                for (Object o : new ArrayList<Object>(stock)) {
                    if (!(o instanceof String)) continue;
                    ItemStack item = decode((String)o);
                    if (item != null) { Map<Integer, ItemStack> leftover = p.getInventory().addItem(item); for (ItemStack rem : leftover.values()) p.getWorld().dropItemNaturally(p.getLocation(), rem); claimed++; }
                }
                data.set(path(id) + ".stock", new ArrayList<String>());
                saveData();
                p.sendMessage(PREFIX + "Claimed " + claimed + " stored payment stack(s) from " + data.getString(path(id)+".name") + ".");
            } else sender.sendMessage(PREFIX + "Usage: /vnpc create <name> | edit <name> | claim <name>");
            return true;
        }
        return false;
    }
    private String join(String[] a, int start) { StringBuilder b = new StringBuilder(); for(int i=start;i<a.length;i++){if(i>start)b.append(' ');b.append(a[i]);} return b.toString(); }
    private UUID findVillager(String name) {
        for (Map.Entry<UUID,String> e : villagerNames.entrySet()) if (e.getValue().equalsIgnoreCase(name)) return e.getKey();
        return null;
    }
    private void openEditor(Player p, UUID id) {
        Inventory inv = Bukkit.createInventory(null, 27, "CAAV Edit: " + data.getString(path(id)+".name"));
        inv.setItem(10, named(Material.GOLD_INGOT, "Set Cost", "Hold the payment item in your main hand", "Then click this button."));
        inv.setItem(12, named(Material.DIAMOND, "Set Reward", "Hold the reward item in your main hand", "Then click this button."));
        inv.setItem(14, named(Material.EMERALD, "Save Trade", "Adds the cost/reward pair as a trade."));
        inv.setItem(16, named(Material.BOOK, "List Trades", "Click to view configured trades."));
        inv.setItem(22, named(Material.BARRIER, "Close"));
        p.openInventory(inv);
    }
    @EventHandler public void onInteract(PlayerInteractEvent e) {
        if (e.getAction() != Action.RIGHT_CLICK_BLOCK) return;
        Player p = e.getPlayer(); ItemStack hand = e.getItem();
        if (isNamed(hand, "CAAV Area Selector") && p.hasPermission("caav.admin")) {
            List<Location> points = selections.get(p.getUniqueId());
            if (points == null) return;
            e.setCancelled(true);
            points.add(e.getClickedBlock().getLocation());
            p.sendMessage(PREFIX + "Corner " + points.size() + "/4 selected at " + e.getClickedBlock().getX()+", "+e.getClickedBlock().getY()+", "+e.getClickedBlock().getZ()+".");
            if (points.size() == 4) {
                String id = UUID.randomUUID().toString();
                List<String> encoded = new ArrayList<String>();
                for (Location l : points) encoded.add(l.getWorld().getName()+","+l.getBlockX()+","+l.getBlockY()+","+l.getBlockZ());
                data.set("areas."+id+".points", encoded); data.set("areas."+id+".world", p.getWorld().getName());
                saveData(); selections.remove(p.getUniqueId());
                p.sendMessage(PREFIX + ChatColor.GREEN + "Area protected! The four corners define the protected polygon.");
            }
            return;
        }
        // Custom villager trade GUI opens on right-click.
        if (e.getClickedBlock() != null) {
            for (Entity entity : p.getNearbyEntities(4, 4, 4)) {
                if (entity instanceof Villager && entity.getLocation().distanceSquared(p.getLocation()) < 16) {
                    Villager v = (Villager)entity;
                    String n = data.getString(path(v.getUniqueId())+".name");
                    if (n != null) { e.setCancelled(true); openTrades(p, v.getUniqueId()); return; }
                }
            }
        }
    }
    private void openTrades(Player p, UUID id) {
        List<?> trades = data.getList(path(id)+".trades", new ArrayList<Object>());
        Inventory inv = Bukkit.createInventory(null, 54, "CAAV Shop: " + data.getString(path(id)+".name"));
        int slot = 0;
        for (Object obj : trades) {
            if (slot >= 45 || !(obj instanceof String)) break;
            String[] pair = ((String)obj).split("\\|", 2);
            if (pair.length != 2) continue;
            ItemStack reward = decode(pair[1]); ItemStack cost = decode(pair[0]);
            if (reward == null || cost == null) continue;
            ItemStack display = reward.clone(); ItemMeta meta = display.getItemMeta();
            meta.setLore(Arrays.asList(ChatColor.GRAY+"Cost: "+cost.getAmount()+"x "+cost.getType(), ChatColor.GREEN+"Click to buy"));
            display.setItemMeta(meta); inv.setItem(slot++, display);
        }
        p.openInventory(inv);
        tradeAction.put(p.getUniqueId(), "shop:" + id.toString());
    }
    @EventHandler public void onInventoryClick(InventoryClickEvent e) {
        if (!(e.getWhoClicked() instanceof Player)) return;
        Player p = (Player)e.getWhoClicked(); String title = e.getView().getTitle();
        if (title.startsWith("CAAV Edit: ")) {
            e.setCancelled(true);
            String raw = editingVillager.get(p.getUniqueId()); if (raw == null) return;
            UUID id = UUID.fromString(raw);
            if (e.getCurrentItem() == null) return;
            if (isNamed(e.getCurrentItem(), "Set Cost")) {
                ItemStack held = p.getInventory().getItemInMainHand();
                if (held == null || held.getType() == Material.AIR) { p.sendMessage(PREFIX+ChatColor.RED+"Hold the payment item and its quantity in your main hand first."); return; }
                data.set(path(id)+".pendingCost."+p.getUniqueId(), encode(held));
                saveData(); p.sendMessage(PREFIX+ChatColor.GREEN+"Payment saved: "+held.getAmount()+"x "+held.getType()+". Reopen the editor and set the reward.");
                openEditor(p,id);
            }
            else if (isNamed(e.getCurrentItem(), "Set Reward")) {
                ItemStack held = p.getInventory().getItemInMainHand();
                if (held == null || held.getType() == Material.AIR) { p.sendMessage(PREFIX+ChatColor.RED+"Hold the reward item and its quantity in your main hand first."); return; }
                data.set(path(id)+".pendingReward."+p.getUniqueId(), encode(held));
                saveData(); p.sendMessage(PREFIX+ChatColor.GREEN+"Reward saved: "+held.getAmount()+"x "+held.getType()+". Reopen the editor and save the trade.");
                openEditor(p,id);
            }
            else if (isNamed(e.getCurrentItem(), "Save Trade")) {
                ItemStack cost = p.getInventory().getItemInMainHand();
                String pendingCost = data.getString(path(id)+".pendingCost."+p.getUniqueId());
                String pendingReward = data.getString(path(id)+".pendingReward."+p.getUniqueId());
                if (pendingCost == null || pendingReward == null) { p.sendMessage(PREFIX+ChatColor.RED+"Set cost and reward first. Hold each item and click its button."); return; }
                List<String> trades = data.getStringList(path(id)+".trades");
                trades.add(pendingCost+"|"+pendingReward); data.set(path(id)+".trades", trades);
                data.set(path(id)+".pendingCost."+p.getUniqueId(), null); data.set(path(id)+".pendingReward."+p.getUniqueId(), null);
                saveData(); p.sendMessage(PREFIX+ChatColor.GREEN+"Trade saved."); openEditor(p,id);
            } else if (isNamed(e.getCurrentItem(), "List Trades")) {
                List<String> ts = data.getStringList(path(id)+".trades");
                p.sendMessage(PREFIX+"Configured trades: "+ts.size());
                for (String t : ts) { String[] parts=t.split("\\|",2); if(parts.length==2) p.sendMessage(ChatColor.GRAY+"Cost "+decode(parts[0]).getAmount()+"x "+decode(parts[0]).getType()+" -> "+decode(parts[1]).getAmount()+"x "+decode(parts[1]).getType()); }
            } else if (isNamed(e.getCurrentItem(), "Close")) p.closeInventory();
            return;
        }
        if (title.startsWith("CAAV Shop: ")) {
            e.setCancelled(true);
            String action = tradeAction.get(p.getUniqueId());
            if (action == null || !action.startsWith("shop:") || e.getRawSlot() >= 45 || e.getRawSlot() < 0) return;
            UUID id = UUID.fromString(action.substring(5));
            List<String> trades = data.getStringList(path(id)+".trades");
            int index = e.getRawSlot(); if (index >= trades.size()) return;
            String[] pair = trades.get(index).split("\\|",2); if(pair.length!=2) return;
            ItemStack cost=decode(pair[0]), reward=decode(pair[1]); if(cost==null||reward==null)return;
            if (!hasItems(p, cost)) { p.sendMessage(PREFIX+ChatColor.RED+"You don't have enough "+cost.getType()+"."); return; }
            removeItems(p,cost);
            Map<Integer,ItemStack> leftovers=p.getInventory().addItem(reward.clone());
            for(ItemStack rem:leftovers.values()) p.getWorld().dropItemNaturally(p.getLocation(),rem);
            List<String> stock=data.getStringList(path(id)+".stock"); stock.add(encode(cost)); data.set(path(id)+".stock",stock); saveData();
            p.sendMessage(PREFIX+ChatColor.GREEN+"Trade completed."); return;
        }
    }
    @EventHandler public void onInventoryClose(InventoryCloseEvent e) {
        if (e.getView().getTitle().startsWith("CAAV Shop: ")) tradeAction.remove(e.getPlayer().getUniqueId());
    }
    private boolean hasItems(Player p, ItemStack wanted) {
        int amount=0; for(ItemStack it:p.getInventory().getContents()) if(it!=null && it.getType()==wanted.getType() && it.getDurability()==wanted.getDurability()) amount+=it.getAmount();
        return amount>=wanted.getAmount();
    }
    private void removeItems(Player p, ItemStack wanted) {
        int left=wanted.getAmount(); ItemStack[] contents=p.getInventory().getContents();
        for(int i=0;i<contents.length && left>0;i++){ItemStack it=contents[i]; if(it!=null && it.getType()==wanted.getType() && it.getDurability()==wanted.getDurability()){int take=Math.min(left,it.getAmount());it.setAmount(it.getAmount()-take);left-=take;if(it.getAmount()<=0)contents[i]=null;}}
        p.getInventory().setContents(contents);
    }
    private String encode(ItemStack it) { return it.getType().name()+","+it.getDurability()+","+it.getAmount(); }
    private ItemStack decode(String s) {
        try { String[] a=s.split(","); return new ItemStack(Material.valueOf(a[0]), Integer.parseInt(a[2]), (short)Integer.parseInt(a[1])); } catch(Exception ex) { return null; }
    }
    private boolean inside(double x, double z, List<Location> points) {
        boolean c=false; int n=points.size();
        for(int i=0,j=n-1;i<n;j=i++) { double xi=points.get(i).getX()+0.5, zi=points.get(i).getZ()+0.5, xj=points.get(j).getX()+0.5, zj=points.get(j).getZ()+0.5;
            if(((zi>z)!=(zj>z)) && (x<(xj-xi)*(z-zi)/(zj-zi)+xi)) c=!c;
        } return c;
    }
    private List<Location> areaPoints(String area) {
        List<Location> out=new ArrayList<Location>();
        for(String s:data.getStringList(area+".points")) { try { String[] a=s.split(","); World w=Bukkit.getWorld(a[0]); if(w!=null) out.add(new Location(w,Integer.parseInt(a[1]),Integer.parseInt(a[2]),Integer.parseInt(a[3]))); }catch(Exception ignored){} }
        return out;
    }
    private boolean protectedAt(Location loc) {
        if(loc==null||loc.getWorld()==null)return false;
        ConfigurationSection sec=data.getConfigurationSection("areas"); if(sec==null)return false;
        for(String id:sec.getKeys(false)) {
            if(!data.getString("areas."+id+".world","").equals(loc.getWorld().getName()))continue;
            List<Location> pts=areaPoints("areas."+id);
            if(pts.size()>=3 && inside(loc.getBlockX()+0.5,loc.getBlockZ()+0.5,pts)) return true;
        } return false;
    }
    @EventHandler public void onBlockBreak(BlockBreakEvent e) { if(protectedAt(e.getBlock().getLocation())&&!e.getPlayer().hasPermission("caav.admin")){e.setCancelled(true);e.getPlayer().sendMessage(PREFIX+ChatColor.RED+"This area is protected.");} }
    @EventHandler public void onBlockPlace(BlockPlaceEvent e) { if(protectedAt(e.getBlock().getLocation())&&!e.getPlayer().hasPermission("caav.admin")){e.setCancelled(true);e.getPlayer().sendMessage(PREFIX+ChatColor.RED+"This area is protected.");} }
    @EventHandler public void onInteractProtected(PlayerInteractEvent e) {
        if(e.getClickedBlock()!=null && protectedAt(e.getClickedBlock().getLocation()) && !e.getPlayer().hasPermission("caav.admin")) {
            // Do not block the custom area selector, and let entity interactions pass through.
            if(!isNamed(e.getItem(),"CAAV Area Selector")) e.setCancelled(true);
        }
    }
    @EventHandler public void onBucket(PlayerBucketEmptyEvent e) { if(protectedAt(e.getBlockClicked().getRelative(e.getBlockFace()).getLocation())&&!e.getPlayer().hasPermission("caav.admin"))e.setCancelled(true); }
    @EventHandler public void onBucketFill(PlayerBucketFillEvent e) { if(protectedAt(e.getBlockClicked().getLocation())&&!e.getPlayer().hasPermission("caav.admin"))e.setCancelled(true); }
    @EventHandler public void onEntityInteract(PlayerInteractEntityEvent e) {
        if(e.getRightClicked() instanceof Villager) {
            Villager v=(Villager)e.getRightClicked();
            if(data.getString(path(v.getUniqueId())+".name")!=null) { e.setCancelled(true); openTrades(e.getPlayer(),v.getUniqueId()); }
        }
    }
}
