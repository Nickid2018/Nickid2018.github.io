这周的快照21w13a发布了，看看都有啥吧！

版本对照（21w11a->21w13a）：https://paste.ubuntu.com/p/WWZxXbx3Tf/

<h2>说在前面</h2>

这个版本开始，原版代码的阅读和直接修改可能会更加便捷——Mojang对一些常量进行了弃用和增加（导致了至少2700个类被修改，详见版本对照表），各个包下也增加了package-info，还加入了有关于混淆的注释。下面是在SharedConstants中具体的表现：

1.弃用的字段

	@Deprecated
	public static final boolean SNAPSHOT = true; // 是否为快照版本
	@Deprecated
	public static final int WORLD_VERSION = 2705; // 世界版本
	@Deprecated
	public static final String VERSION_STRING = "21w13a"; // 版本号
	@Deprecated
	public static final String RELEASE_TARGET = "1.17"; // 发行版本
	@Deprecated
	public static final int RELEASE_NETWORK_PROTOCOL_VERSION = 755; //发行网络版本协议号
	@Deprecated
	public static final int SNAPSHOT_NETWORK_PROTOCOL_VERSION = 20; // 快照网络版本协议号
	@Deprecated
	public static final int RESOURCE_PACK_FORMAT = 7; // 资源包版本
	@Deprecated
	public static final int DATA_PACK_FORMAT = 7; // 数据包版本
	
2.增加的字段

	public static final boolean NEW_WORLD_GENERATION = true;
	public static final boolean EXTENDED_WORLD_HEIGHT = true;
	public static final boolean USE_NEW_RENDERSYSTEM = false; // 开闭新渲染系统？
	public static final boolean MULTITHREADED_RENDERING = false; // 多线程渲染？
	public static final boolean FIX_TNT_DUPE = false; // TNT 复制修复？
	public static final boolean ENABLE_SNOOPER = false;
	/* 调试常量 */
	public static final boolean USE_DEBUG_FEATURES = false;
	public static final boolean DEBUG_HOTKEYS = false;
	public static final boolean DEBUG_RENDER = false;
	public static final boolean DEBUG_PATHFINDING = false;
	public static final boolean DEBUG_WATER = false;
	public static final boolean DEBUG_HEIGHTMAP = false;
	public static final boolean DEBUG_COLLISION = false;
	public static final boolean DEBUG_SHAPES = false;
	public static final boolean DEBUG_NEIGHBORSUPDATE = false;
	public static final boolean DEBUG_STRUCTURES = false;
	public static final boolean DEBUG_LIGHT = false;
	public static final boolean DEBUG_WORLDGENATTEMPT = false;
	public static final boolean DEBUG_SOLID_FACE = false;
	public static final boolean DEBUG_CHUNKS = false;
	public static final boolean DEBUG_GAME_EVENT_LISTENERS = false;
	public static final boolean DEBUG_DUMP_TEXTURE_ATLAS = false;
	public static final boolean DEBUG_DUMP_INTERPOLATED_TEXTURE_FRAMES = false;
	public static final boolean DEBUG_STRUCTURE_EDIT_MODE = false;
	public static final boolean DEBUG_SAVE_STRUCTURES_AS_SNBT = false;
	public static final boolean DEBUG_SYNCHRONOUS_GL_LOGS = false;
	public static final boolean DEBUG_VERBOSE_SERVER_EVENTS = false;
	public static final boolean DEBUG_NAMED_RUNNABLES = false;
	public static final boolean DEBUG_GOAL_SELECTOR = false;
	public static final boolean DEBUG_VILLAGE_SECTIONS = false;
	public static final boolean DEBUG_BRAIN = false;
	public static final boolean DEBUG_BEES = false;
	public static final boolean DEBUG_RAIDS = false;
	public static final boolean DEBUG_BLOCK_BREAK = false;
	public static final boolean DEBUG_RESOURCE_LOAD_TIMES = false;
	public static final boolean DEBUG_MONITOR_TICK_TIMES = false;
	public static final boolean DEBUG_KEEP_JIGSAW_BLOCKS_DURING_STRUCTURE_GEN = false;
	public static final boolean DEBUG_DONT_SAVE_WORLD = false;
	public static final boolean DEBUG_LARGE_DRIPSTONE = false;
	public static final boolean DEBUG_PACKET_SERIALIZATION = false;
	public static final boolean DEBUG_CARVERS = false;
	public static final boolean DEBUG_SMALL_SPAWN = false;
	public static final boolean DEBUG_DISABLE_LIQUID_SPREADING = false;
	public static final boolean DEBUG_ONLY_GENERATE_HALF_THE_WORLD = false;
	public static final boolean DEBUG_DISABLE_WATER_GENERATION = false;
	public static final boolean DEBUG_DISABLE_AQUIFERS = false;
	public static final boolean DEBUG_DISABLE_NOISE_CAVES = false;
	public static final boolean DEBUG_DISABLE_SURFACE = false;
	public static final boolean DEBUG_DISABLE_CARVERS = false;
	public static final boolean DEBUG_DISABLE_STRUCTURES = false;
	public static final boolean DEBUG_DISABLE_FEATURES = false;
	public static final boolean INGAME_DEBUG_OUTPUT = false;
	public static final boolean DEBUG_SUBTITLES = false;
	public static final boolean DEBUG_WORLD_RECREATE = false;
	public static final boolean DEBUG_SHOW_SERVER_DEBUG_VALUES = false;
	public static final boolean DEBUG_STORE_CHUNK_STACKTRACES = false;
	public static final int FAKE_MS_LATENCY = 0;
	public static final int FAKE_MS_JITTER = 0;
	public static final Level NETTY_LEAK_DETECTION;
	public static final boolean COMMAND_STACK_TRACES = false;
	/* 世界设置 */
	public static final int DEFAULT_MINECRAFT_PORT = 25565;
	public static final float RAIN_THRESHOLD = 0.15F;
	public static final long MAXIMUM_TICK_TIME_NANOS;
	public static final int WORLD_RESOLUTION = 16;
	public static final int MAX_CHAT_LENGTH = 256;
	public static final int MAX_COMMAND_LENGTH = 32500;
	public static final int TICKS_PER_SECOND = 20;
	public static final int TICKS_PER_MINUTE = 1200;
	public static final int TICKS_PER_GAME_DAY = 24000;
	public static final float AVERAGE_GAME_TICKS_PER_RANDOM_TICK_PER_BLOCK = 1365.3334F;
	public static final float AVERAGE_RANDOM_TICKS_PER_BLOCK_PER_MINUTE = 0.87890625F;
	public static final float AVERAGE_RANDOM_TICKS_PER_BLOCK_PER_GAME_DAY = 17.578125F;
	
目前尚不清楚这些字段的用处，但是也许对理解Mojang的代码有好处。

<h2>一.山羊</h2>

山羊是这个版本新增加的一种生物，现在没有自然生成。它的主要特性是跳跃、挤奶，现在冲撞AI还没有加入，但是已经写了"isScreamingGoat"这个字段，应该在下个版本就能加入。

![山羊](/resources/20210404/goats.png)

先说说跳跃的特性。山羊的跳跃由两个类控制：LongJumpMidJump和LongJumpToRandomPos。

	// 初始化跳跃AI
	private static void initLongJumpActivity(Brain<Goat> brain) {
		brain.addActivityWithConditions(Activity.LONG_JUMP,
				ImmutableList.of((Object) Pair.of((Object) 0, (Object) new LongJumpMidJump(TIMES_BETWEEN_LONG_JUMPS)),
						(Object) Pair.of((Object) 1,
								(Object) new LongJumpToRandomPos(TIMES_BETWEEN_LONG_JUMPS, 5, 5, 1.5f))),
				(Set) ImmutableSet.of(
						(Object) Pair.of((Object) MemoryModuleType.TEMPTING_PLAYER, (Object) MemoryStatus.VALUE_ABSENT),
						(Object) Pair.of((Object) MemoryModuleType.BREED_TARGET, (Object) MemoryStatus.VALUE_ABSENT),
						(Object) Pair.of((Object) MemoryModuleType.WALK_TARGET, (Object) MemoryStatus.VALUE_ABSENT),
						(Object) Pair.of((Object) MemoryModuleType.LONG_JUMP_COOLDOWN_TICKS,
								(Object) MemoryStatus.VALUE_ABSENT)));
	}
	
每次尝试跳跃间隔是600-1200游戏刻。在跳跃之前，山羊会检查脚下方块，如果是蜂蜜块，则不会跳跃。之后，山羊会在周围9x9x9范围内寻找可以进行跳跃的方块，生成一个可跳跃方块列表。在生成跳跃方块列表后，山羊会随机选取已经生成的可到达位置作为要跳跃到的位置并再次检查这个位置是否可被寻路，寻路最大距离为7格。若寻路条件判断失败，在下一随机刻还会再次随机选取判断，一共尝试20次。如果成功，山羊将面对将要跳向的方块，也就是"准备跳跃"。"准备跳跃"时长40tick，在此时间内山羊不会再次检查方块是否可跳跃，所以在此期间破坏将要跳向的方块不会导致此AI取消。40tick结束后，山羊跳跃，跳跃最大速度为1.5m/tick，摩擦失效，跳跃提升对此无作用。落地时，摩擦重新生效，速度变为之前的0.1倍。

	// 开始AI运作
	protected void start(ServerLevel serverLevel, Mob mob, long l) {
		this.chosenJump = Optional.empty();
		this.findJumpTries = 20;
		this.jumpCandidates.clear();
		this.initialPosition = Optional.of(mob.position());
		BlockPos blockPos = mob.blockPosition();
		int n = blockPos.getX();
		int n2 = blockPos.getY();
		int n3 = blockPos.getZ();
		Iterable iterable = BlockPos.betweenClosed(n - this.maxLongJumpWidth,
				n2 - this.maxLongJumpHeight, n3 - this.maxLongJumpWidth,
				n + this.maxLongJumpWidth, n2 + this.maxLongJumpHeight,
				n3 + this.maxLongJumpWidth);
		PathNavigation pathNavigation = mob.getNavigation();
		for (BlockPos blockPos2 : iterable) {
			double d = blockPos2.distSqr(blockPos);
			if (n == blockPos2.getX() && n3 == blockPos2.getZ() || !pathNavigation.isStableDestination(blockPos2)
					|| mob.getPathfindingMalus(WalkNodeEvaluator.getBlockPathTypeStatic(mob.level,blockPos2.mutable())) != 0.0f)
				continue;
			Optional<Vec3> optional = this.calculateOptimalJumpVector(mob, Vec3.atCenterOf(blockPos2));
			optional.ifPresent(vec3 -> this.jumpCandidates
					.add(new PossibleJump(new BlockPos(blockPos2), vec3, Mth.ceil(d))));
		}
	}
	// AI的游戏刻运行
	protected void tick(ServerLevel serverLevel, Mob mob, long l) {
		if (this.chosenJump.isPresent()) {
			if (l - this.prepareJumpStart >= 40L) {
				mob.yRot = mob.yBodyRot;
				mob.setDiscardFriction(true);
				mob.setDeltaMovement(this.chosenJump.get().getJumpVector());
				mob.getBrain().setMemory(MemoryModuleType.LONG_JUMP_MID_JUMP, (Object) true);
			}
		} else {
			--this.findJumpTries;
			Optional optional = WeighedRandom.getRandomItem(serverLevel.random, this.jumpCandidates);
			if (optional.isPresent()) {
				this.jumpCandidates.remove(optional.get());
				mob.getBrain().setMemory(MemoryModuleType.LOOK_TARGET,
						(Object) new BlockPosTracker(((PossibleJump) optional.get()).getJumpTarget()));
				PathNavigation pathNavigation = mob.getNavigation();
				Path path = pathNavigation.createPath(((PossibleJump) optional.get()).getJumpTarget(), 0, 7);
				if (path == null || !path.canReach()) { 
					this.chosenJump = optional;
					this.prepareJumpStart = l;
				}
			}
		}
	}

那么山羊为什么从这么高的位置摔下不会扣血呢？答案是：覆写了calculateFallDamage。山羊在计算下落高度时会减掉10格，所以即使跳的很高也不会摔掉血。

	// 计算下落距离
	protected int calculateFallDamage(float f, float f2) {
		return super.calculateFallDamage(f, f2) - 10;
	}

另一个特性是挤奶，这个类似于牛。

	// 当玩家右键生物进行互动时调用此方法
	public InteractionResult mobInteract(Player player, InteractionHand interactionHand) {
		ItemStack itemStack = player.getItemInHand(interactionHand);
		if (itemStack.is(Items.BUCKET) && !this.isBaby()) {
			player.playSound(this.getMilkingSound(), 1.0f, 1.0f);
			ItemStack itemStack2 = ItemUtils.createFilledResult(itemStack, player, Items.MILK_BUCKET.getDefaultInstance());
			player.setItemInHand(interactionHand, itemStack2);
			return InteractionResult.sidedSuccess(this.level.isClientSide);
		}
		return super.mobInteract(player, interactionHand);
	}
	
<h2>二.光源方块</h2>

光源方块本身是基岩版的专属，在此版本也加入了Java版，它是一种看不见的方块，不阻碍天空光，类似屏障只能手持同类物品才能看到，能散发出指定强度的方块光，同时也是一个含水方块。

![光源方块](/resources/20210404/light.png)

它的获取和同样身为"OP方块"的命令方块、结构方块、拼图方块、结构空位一样需要用give指令。获取它需要输入`/give @s minecraft:light`。如果要获得指定亮度的光源方块物品（默认为15级亮度方块），就要输入`/give @s minecraft:light{"BlockStateTag":{"level":<亮度0-15>}}`。同样，对一个已经放置的光源方块中建选取也能获得物品（必须手持另一个光源方块）。

	// 中键选取方块时调用
	public ItemStack getCloneItemStack(BlockGetter blockGetter, BlockPos blockPos, BlockState blockState) {
		ItemStack itemStack = super.getCloneItemStack(blockGetter, blockPos, blockState);
		CompoundTag compoundTag = new CompoundTag();
		compoundTag.putString(LEVEL.getName(), String.valueOf(blockState.getValue(LEVEL)));
		itemStack.addTagElement("BlockStateTag", (Tag) compoundTag);
		return itemStack;
	}
	
在平常状况下，我们无法碰到光源方块，因为它平常状态下没有包围盒。但是当我们手持光源方块物品时，它就能被点中，并且还能像屏障一样显示。

	// 当玩家看向这个方块时，视线计算碰撞使用此方法
	public VoxelShape getShape(BlockState blockState, BlockGetter blockGetter, BlockPos blockPos,
			CollisionContext collisionContext) {
		return collisionContext.isHoldingItem(Items.LIGHT) ? Shapes.block() : Shapes.empty();
	}
	
当我们右键它时，它会更改它的亮度，从0-15之间轮回。

	// 当玩家以右键与方块交互时，调用此方法
	public InteractionResult use(BlockState blockState, Level level, BlockPos blockPos, Player player,
			InteractionHand interactionHand, BlockHitResult blockHitResult) {
		if (!level.isClientSide) {
			level.setBlock(blockPos, blockState.cycle(LEVEL), 2 /* 注:这个2(0b10)是更新标识 */);
			return InteractionResult.SUCCESS;
		}
		return InteractionResult.CONSUME;
	}

<h2>三.细雪方块的修改</h2>

细雪方块在这个版本进行了一些修改，如下：

1.当着火的生物实体进入细雪内，火会熄灭并且这个细雪方块消失。

![骷髅着火走入细雪中，细雪消失](/resources/20210404/clear_fire.png)

2.下落速度变为1.5（之前是0.99）

	// 当实体进入方块内部，调用此方法
	public void entityInside(BlockState blockState, Level level, BlockPos blockPos, Entity entity) {
		if (!(entity instanceof LivingEntity) || ((LivingEntity) entity).getFeetBlockState().is(Blocks.POWDER_SNOW)) {
			entity.makeStuckInBlock(blockState, new Vec3(0.8999999761581421, 1.5, 0.8999999761581421));
		}
		entity.setIsInPowderSnow(true);
		if (entity.isOnFire()) {
			level.setBlockAndUpdate(blockPos, Blocks.AIR.defaultBlockState());
			level.addDestroyBlockEffect(blockPos, blockState);
		}
		if (level.isClientSide) {
			entity.clearFire();
		} else {
			entity.setSharedFlagOnFire(false);
		}
		if (!entity.isSpectator() && (entity.xOld != entity.getX() || entity.zOld != entity.getZ())
				&& level.random.nextBoolean()) {
			PowderSnowBlock.spawnPowderSnowParticles(level,
					new Vec3(entity.getX(), (double) blockPos.getY(), entity.getZ()));
		}
	}
	
3.实体冻伤开始时间变为140tick。完全冰冻后，每40tick受到1点伤害；对于烈焰人、岩浆怪、炽足兽这种带有FREEZE\_HURTS\_EXTRA\_TYPES的生物为5点伤害。

	// aiStep中的一段代码
	boolean bl = this.getType().is(EntityTypeTags.FREEZE_HURTS_EXTRA_TYPES);
	if (!this.level.isClientSide) {
		n = this.getTicksFrozen();
		if (this.isInPowderSnow && this.canFreeze()) {
			this.setTicksFrozen(Math.min(this.getTicksRequiredToFreeze(), n + 1));
		} else {
			this.setTicksFrozen(Math.max(0, n - 2));
		}
	}
	removeFrost();
	tryAddFrost();
	if (!level.isClientSide && tickCount % 40 == 0 && this.isFullyFrozen() && this.canFreeze()) {
		n = bl ? 5 : 1;
		this.hurt(DamageSource.FREEZE, n);
	}
	
4.骷髅转变为流浪者从进入细雪的140tick开始，再经过300tick转变为流浪者。

![骷髅转化为流浪者](/resources/20210404/ske_stray.png)

	// Skeleton类中的一部分代码，执行游戏刻
	public void tick() {
		if (!this.level.isClientSide && this.isAlive() && !this.isNoAi()) {
			if (this.isFreezeConverting()) {
				--this.conversionTime;
				if (this.conversionTime < 0) {
					this.doFreezeConversion();
				}
			} else if (this.isInPowderSnow) {
				++this.inPowderSnowTime;
				if (this.inPowderSnowTime >= 140) {
					this.startFreezeConversion(300);
				}
			} else {
				this.inPowderSnowTime = -1;
			}
		}
		super.tick();
	}

<h2>四.美西螈的修改</h2>

美西螈的一些bug在这个版本中被修复（即使MC-208626还在坚持着）

1.美西螈桶现在能保存美西螈的信息，特有的NBT标签为variant（变种）和age（年龄）。可是，Bugjump忘加了美西螈CustomName的保存，所以将美西螈装到桶里再放出来命名丢失。（各种鱼桶同理）

	// 用桶接
	public InteractionResult mobInteract(Player player, InteractionHand interactionHand) {
		return Bucketable.bucketMobPickup(player, interactionHand, this)
				.orElse(super.mobInteract(player, interactionHand));
	}
	// 保存到桶中的额外数据
	public void saveToBucketTag(ItemStack itemStack) {
		Bucketable.saveDefaultDataToBucketTag(this, itemStack);
		CompoundTag compoundTag = itemStack.getOrCreateTag();
		compoundTag.putInt(VARIANT_TAG, this.getVariant().getId());
		compoundTag.putInt("Age", this.getAge());
	}
	// 恢复额外数据
	public void loadFromBucketTag(CompoundTag compoundTag) {
		Bucketable.loadDefaultDataFromBucketTag(this, compoundTag);
		this.setVariant(Variant.BY_ID[compoundTag.getInt(VARIANT_TAG)]);
		if (compoundTag.contains("Age")) {
			this.setAge(compoundTag.getInt("Age"));
		}
	}
	
2.修复了无法清除挖掘疲劳和给予生命恢复的bug

	// AxolotlAi调用了此方法
	public static void applySupportingEffects(Player player) {
		MobEffectInstance mobEffectInstance = player.getEffect(MobEffects.REGENERATION);
		int n = 100 + (mobEffectInstance != null ? mobEffectInstance.getDuration() : 0);
		player.addEffect(new MobEffectInstance(MobEffects.REGENERATION, n, 0));
		player.removeEffect(MobEffects.DIG_SLOWDOWN);
	}
	
3.击杀非敌对生物获得2400tick冷却时间