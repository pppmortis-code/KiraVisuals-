name: Build KiroVisuals
on: [push, workflow_dispatch]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 11
      - uses: gradle/actions/setup-gradle@v3
        with:
          gradle-version: 7.3.3
      - name: Create project files
        run: |
          set -e
          mkdir -p KiroVisuals && cd KiroVisuals
          cat > 'build.gradle' << 'KIRO_EOF'
          plugins {
              id 'fabric-loom' version '0.10-SNAPSHOT'
          }

          sourceCompatibility = JavaVersion.VERSION_1_8
          targetCompatibility = JavaVersion.VERSION_1_8

          archivesBaseName = project.archives_base_name
          version = project.mod_version
          group = project.maven_group

          dependencies {
              minecraft "com.mojang:minecraft:${project.minecraft_version}"
              mappings "net.fabricmc:yarn:${project.yarn_mappings}:v2"
              modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
              modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
          }

          processResources {
              inputs.property "version", project.version
              filesMatching("fabric.mod.json") {
                  expand "version": project.version
              }
          }

          tasks.withType(JavaCompile).configureEach {
              it.options.encoding = "UTF-8"
              it.options.release = 8
          }
          KIRO_EOF
          cat > 'gradle.properties' << 'KIRO_EOF'
          org.gradle.jvmargs=-Xmx1G

          # Minecraft 1.16.5 / Fabric
          minecraft_version=1.16.5
          yarn_mappings=1.16.5+build.10
          loader_version=0.11.7
          fabric_version=0.42.0+1.16

          # Mod
          mod_version=1.0.0
          maven_group=dev.kiro
          archives_base_name=kirovisuals
          KIRO_EOF
          cat > 'settings.gradle' << 'KIRO_EOF'
          pluginManagement {
              repositories {
                  maven {
                      name = 'Fabric'
                      url = 'https://maven.fabricmc.net/'
                  }
                  mavenCentral()
                  gradlePluginPortal()
              }
          }

          rootProject.name = 'KiroVisuals'
          KIRO_EOF
          mkdir -p 'src/main/java/dev/kiro/visuals'
          cat > 'src/main/java/dev/kiro/visuals/KiroVisuals.java' << 'KIRO_EOF'
          package dev.kiro.visuals;

          import dev.kiro.visuals.config.VisualsConfig;
          import dev.kiro.visuals.gui.KiroVisualsScreen;
          import dev.kiro.visuals.hud.HudOverlay;
          import dev.kiro.visuals.render.AuraRenderer;
          import dev.kiro.visuals.render.ParticleRenderer;
          import dev.kiro.visuals.render.RenderUtil;
          import net.fabricmc.api.ClientModInitializer;
          import net.fabricmc.fabric.api.client.event.lifecycle.v1.ClientLifecycleEvents;
          import net.fabricmc.fabric.api.client.event.lifecycle.v1.ClientTickEvents;
          import net.fabricmc.fabric.api.client.keybinding.v1.KeyBindingHelper;
          import net.fabricmc.fabric.api.client.rendering.v1.HudRenderCallback;
          import net.fabricmc.fabric.api.client.rendering.v1.WorldRenderContext;
          import net.fabricmc.fabric.api.client.rendering.v1.WorldRenderEvents;
          import net.minecraft.client.MinecraftClient;
          import net.minecraft.client.options.KeyBinding;
          import net.minecraft.client.render.RenderLayer;
          import net.minecraft.client.render.VertexConsumer;
          import net.minecraft.client.render.VertexConsumerProvider;
          import net.minecraft.client.util.InputUtil;
          import net.minecraft.client.util.math.MatrixStack;
          import org.apache.logging.log4j.LogManager;
          import org.apache.logging.log4j.Logger;
          import org.lwjgl.glfw.GLFW;

          /** KiroVisuals entry point (Fabric, client side only). Menu key: Right Shift. */
          public class KiroVisuals implements ClientModInitializer {
              public static final String MOD_ID = "kirovisuals";
              public static final Logger LOGGER = LogManager.getLogger("KiroVisuals");

              public static VisualsConfig config;
              public static KeyBinding openMenu;

              /** Set when a render error happens, so a bug can never spam the log or crash every frame. */
              private static boolean renderBroken;

              @Override
              public void onInitializeClient() {
                  config = VisualsConfig.load();

                  openMenu = KeyBindingHelper.registerKeyBinding(new KeyBinding(
                          "key.kirovisuals.menu", InputUtil.Type.KEYSYM, GLFW.GLFW_KEY_RIGHT_SHIFT, "key.categories.kirovisuals"));

                  ClientTickEvents.END_CLIENT_TICK.register(KiroVisuals::onTick);
                  HudRenderCallback.EVENT.register(HudOverlay::render);
                  WorldRenderEvents.LAST.register(KiroVisuals::onWorldRender);
                  ClientLifecycleEvents.CLIENT_STOPPING.register(client -> config.save());

                  LOGGER.info("KiroVisuals loaded. Press Right Shift in game to open the menu.");
              }

              private static void onTick(MinecraftClient client) {
                  while (openMenu.wasPressed()) {
                      if (client.currentScreen == null && client.player != null) {
                          client.openScreen(new KiroVisualsScreen());
                      }
                  }
                  try {
                      ParticleRenderer.tick(client);
                      AuraRenderer.tick(client);
                  } catch (Throwable t) {
                      LOGGER.error("KiroVisuals tick error, particles cleared", t);
                      ParticleRenderer.clear();
                  }
              }

              private static void onWorldRender(WorldRenderContext ctx) {
                  if (renderBroken || config == null) return;
                  MinecraftClient mc = MinecraftClient.getInstance();
                  if (mc.world == null || mc.player == null) return;
                  try {
                      MatrixStack ms = ctx.matrixStack();
                      float td = ctx.tickDelta();
                      float op = config.opacity();
                      RenderUtil.beginFrame(ctx.camera());

                      VertexConsumerProvider.Immediate buffers = mc.getBufferBuilders().getEntityVertexConsumers();

                      // pass 1: glowing quads (particles, glow, orbs)
                      VertexConsumer quads = buffers.getBuffer(RenderLayer.getLightning());
                      ParticleRenderer.render(ms.peek().getModel(), td, quads, op);
                      AuraRenderer.renderQuads(ms, ctx.camera(), td, quads, mc, op);
                      buffers.draw(RenderLayer.getLightning());

                      // pass 2: lines (aura, rings, outline, symbols, sky stars)
                      VertexConsumer lines = buffers.getBuffer(RenderLayer.getLines());
                      AuraRenderer.renderLines(ms, ctx.camera(), td, lines, mc, op);
                      AuraRenderer.renderSky(ms, td, lines, mc, op);
                      buffers.draw(RenderLayer.getLines());
                  } catch (Throwable t) {
                      renderBroken = true;
                      LOGGER.error("KiroVisuals world rendering disabled after an error", t);
                  }
              }
          }
          KIRO_EOF
          mkdir -p 'src/main/java/dev/kiro/visuals/config'
          cat > 'src/main/java/dev/kiro/visuals/config/Feature.java' << 'KIRO_EOF'
          package dev.kiro.visuals.config;

          import dev.kiro.visuals.gui.VisualsCategory;

          import java.util.Locale;

          /** Description of one menu entry: an ON/OFF toggle or a slider. */
          public final class Feature {
              public final String id;
              public final String name;
              public final String nameLower;
              public final VisualsCategory category;
              public final boolean slider;
              public final boolean defaultOn;
              public final float min;
              public final float max;
              public final float def;
              public final float step;
              public final boolean hue;
              public final String animKey;
              public final String hoverKey;

              private Feature(String id, String name, VisualsCategory category, boolean slider, boolean defaultOn,
                              float min, float max, float def, float step, boolean hue) {
                  this.id = id;
                  this.name = name;
                  this.nameLower = name.toLowerCase(Locale.ROOT);
                  this.category = category;
                  this.slider = slider;
                  this.defaultOn = defaultOn;
                  this.min = min;
                  this.max = max;
                  this.def = def;
                  this.step = step;
                  this.hue = hue;
                  this.animKey = "t:" + id;
                  this.hoverKey = "h:" + id;
              }

              public static Feature toggle(String id, String name, VisualsCategory c, boolean defaultOn) {
                  return new Feature(id, name, c, false, defaultOn, 0f, 1f, 0f, 0f, false);
              }

              public static Feature slider(String id, String name, VisualsCategory c, float min, float max, float def, float step) {
                  return new Feature(id, name, c, true, false, min, max, def, step, false);
              }

              public static Feature hue(String id, String name, VisualsCategory c, float def) {
                  return new Feature(id, name, c, true, false, 0f, 1f, def, 0f, true);
              }
          }
          KIRO_EOF
          mkdir -p 'src/main/java/dev/kiro/visuals/config'
          cat > 'src/main/java/dev/kiro/visuals/config/Features.java' << 'KIRO_EOF'
          package dev.kiro.visuals.config;

          import dev.kiro.visuals.gui.VisualsCategory;

          import java.util.ArrayList;
          import java.util.HashMap;
          import java.util.List;
          import java.util.Map;

          /** Registry of every toggle and slider shown in the menu. */
          public final class Features {
              public static final List<Feature> ALL = new ArrayList<>();
              private static final Map<String, Feature> BY_ID = new HashMap<>();

              private static void add(Feature f) {
                  ALL.add(f);
                  BY_ID.put(f.id, f);
              }

              public static Feature get(String id) {
                  return BY_ID.get(id);
              }

              static {
                  VisualsCategory ch = VisualsCategory.CHARACTER;
                  add(Feature.toggle("custom_aura", "Custom Aura", ch, false));
                  add(Feature.toggle("sakura_particles", "Sakura Particles", ch, false));
                  add(Feature.toggle("blue_flame", "Blue Flame", ch, false));
                  add(Feature.toggle("lightning_aura", "Lightning Aura", ch, false));
                  add(Feature.toggle("energy_ring", "Energy Ring", ch, false));
                  add(Feature.toggle("neon_outline", "Neon Outline", ch, false));
                  add(Feature.toggle("glow_effect", "Glow Effect", ch, false));
                  add(Feature.toggle("floating_symbols", "Floating Symbols", ch, false));
                  add(Feature.toggle("particle_trail", "Particle Trail", ch, false));
                  add(Feature.toggle("magic_circle", "Magic Circle", ch, false));
                  add(Feature.toggle("star_aura", "Star Aura", ch, false));
                  add(Feature.toggle("custom_color_aura", "Custom Color Aura", ch, false));
                  add(Feature.toggle("rainbow_effect", "Rainbow Effect", ch, false));
                  add(Feature.toggle("orbiting_orbs", "Orbiting Orbs", ch, false));
                  add(Feature.toggle("dust_particles", "Dust Particles", ch, false));

                  VisualsCategory wo = VisualsCategory.WORLD;
                  add(Feature.toggle("custom_sky", "Custom Sky", wo, false));
                  add(Feature.toggle("sakura_atmosphere", "Sakura Atmosphere", wo, false));
                  add(Feature.toggle("blue_night", "Blue Night", wo, false));
                  add(Feature.toggle("sunset_mode", "Sunset Mode", wo, false));
                  add(Feature.toggle("bloom_style", "Bloom Style", wo, false));
                  add(Feature.toggle("color_correction", "Color Correction", wo, false));
                  add(Feature.toggle("fog_effects", "Fog Effects", wo, false));
                  add(Feature.toggle("ambient_particles", "Ambient Particles", wo, false));
                  add(Feature.toggle("starry_sky", "Starry Sky", wo, false));
                  add(Feature.toggle("weather_particles", "Weather Particles", wo, false));

                  VisualsCategory hu = VisualsCategory.HUD;
                  add(Feature.toggle("custom_crosshair", "Custom Crosshair", hu, false));
                  add(Feature.hue("crosshair_hue", "Crosshair Color", hu, 0.58f));
                  add(Feature.slider("crosshair_sat", "Crosshair Saturation", hu, 0f, 1f, 0f, 0f));
                  add(Feature.slider("crosshair_size", "Crosshair Size", hu, 2f, 20f, 6f, 1f));
                  add(Feature.slider("crosshair_thickness", "Crosshair Thickness", hu, 1f, 4f, 1f, 1f));
                  add(Feature.toggle("fps_counter", "FPS Counter", hu, true));
                  add(Feature.toggle("coordinates_hud", "Coordinates HUD", hu, true));
                  add(Feature.toggle("direction_hud", "Direction HUD", hu, false));
                  add(Feature.toggle("keystrokes", "Keystrokes", hu, false));
                  add(Feature.toggle("cps_counter", "CPS Counter", hu, false));
                  add(Feature.toggle("armor_hud", "Armor HUD", hu, false));
                  add(Feature.toggle("potion_hud", "Potion HUD", hu, false));
                  add(Feature.toggle("clock", "Clock", hu, false));
                  add(Feature.toggle("session_timer", "Session Timer", hu, false));

                  VisualsCategory se = VisualsCategory.SETTINGS;
                  add(Feature.hue("effect_hue", "Effect Color", se, 0.58f));
                  add(Feature.slider("effect_sat", "Effect Saturation", se, 0f, 1f, 0.85f, 0f));
                  add(Feature.slider("intensity", "Intensity", se, 0.2f, 2f, 1f, 0f));
                  add(Feature.slider("particle_size", "Particle Size", se, 0.4f, 3f, 1f, 0f));
                  add(Feature.slider("anim_speed", "Animation Speed", se, 0.2f, 3f, 1f, 0f));
                  add(Feature.slider("radius", "Radius", se, 0.5f, 3f, 1f, 0f));
                  add(Feature.slider("opacity", "Opacity", se, 0.1f, 1f, 0.85f, 0f));
                  add(Feature.slider("ui_scale", "UI Scale", se, 0.6f, 1.6f, 1f, 0f));
                  add(Feature.hue("ui_hue", "UI Accent", se, 0.60f));
                  add(Feature.toggle("performance_mode", "Performance Mode", se, false));
              }

              private Features() {
              }
          }
          KIRO_EOF
          mkdir -p 'src/main/java/dev/kiro/visuals/config'
          cat > 'src/main/java/dev/kiro/visuals/config/VisualsConfig.java' << 'KIRO_EOF'
          package dev.kiro.visuals.config;

          import com.google.gson.Gson;
          import com.google.gson.GsonBuilder;
          import dev.kiro.visuals.KiroVisuals;
          import net.fabricmc.loader.api.FabricLoader;

          import java.io.Reader;
          import java.io.Writer;
          import java.nio.charset.StandardCharsets;
          import java.nio.file.Files;
          import java.nio.file.Path;
          import java.util.LinkedHashMap;
          import java.util.Map;

          /** Settings storage. Saved to config/kirovisuals.json and restored on the next launch. */
          public final class VisualsConfig {
              private static final Gson GSON = new GsonBuilder().setPrettyPrinting().create();

              public Map<String, Boolean> toggles = new LinkedHashMap<>();
              public Map<String, Float> values = new LinkedHashMap<>();
              public float panelX = -1f;
              public float panelY = -1f;
              public int lastTab = 0;

              private static Path file() {
                  return FabricLoader.getInstance().getConfigDir().resolve("kirovisuals.json");
              }

              public static VisualsConfig load() {
                  VisualsConfig cfg = null;
                  try {
                      Path p = file();
                      if (Files.exists(p)) {
                          try (Reader r = Files.newBufferedReader(p, StandardCharsets.UTF_8)) {
                              cfg = GSON.fromJson(r, VisualsConfig.class);
                          }
                      }
                  } catch (Exception e) {
                      KiroVisuals.LOGGER.error("Could not read config, using defaults", e);
                  }
                  if (cfg == null) {
                      cfg = new VisualsConfig();
                  }
                  cfg.normalize();
                  return cfg;
              }

              /** Fills in missing keys and clamps stored values so that a broken file can never crash the mod. */
              private void normalize() {
                  if (toggles == null) toggles = new LinkedHashMap<>();
                  if (values == null) values = new LinkedHashMap<>();
                  for (Feature f : Features.ALL) {
                      if (f.slider) {
                          Float v = values.get(f.id);
                          values.put(f.id, v == null ? f.def : clamp(v, f.min, f.max));
                      } else if (!toggles.containsKey(f.id)) {
                          toggles.put(f.id, f.defaultOn);
                      }
                  }
                  lastTab = Math.max(0, Math.min(3, lastTab));
              }

              public synchronized void save() {
                  try {
                      Path p = file();
                      Files.createDirectories(p.getParent());
                      try (Writer w = Files.newBufferedWriter(p, StandardCharsets.UTF_8)) {
                          GSON.toJson(this, w);
                      }
                  } catch (Exception e) {
                      KiroVisuals.LOGGER.error("Could not save config", e);
                  }
              }

              /** Restores every toggle and slider to its default value. */
              public void reset() {
                  toggles.clear();
                  values.clear();
                  normalize();
              }

              public boolean on(String id) {
                  Boolean b = toggles.get(id);
                  return b != null && b;
              }

              public void setOn(String id, boolean v) {
                  toggles.put(id, v);
              }

              public float val(String id) {
                  Float v = values.get(id);
                  if (v != null) return v;
                  Feature f = Features.get(id);
                  return f == null ? 0f : f.def;
              }

              public void setVal(String id, float v) {
                  Feature f = Features.get(id);
                  if (f == null) return;
                  if (f.step > 0f) v = Math.round(v / f.step) * f.step;
                  values.put(id, clamp(v, f.min, f.max));
              }

              private static float clamp(float v, float lo, float hi) {
                  return v < lo ? lo : Math.min(v, hi);
              }

              // Shortcuts for the gl
              
