<template>
    <ManagerActivityBar />
	<div class="manager-main-view">
        <CategoryFilterModal />
        <IncorrectGameDirectoryModal />
        <IncorrectSteamDirectoryModal />
        <LaunchArgumentsModal />
        <DependencyStringsModal />
        <SteamInstallationValidationModal />
        <SortModal />
        <LocalFileImportModal :visible="importingLocalMod" @close-modal="importingLocalMod = false" />
        <ProfileCodeExportModal />
        <DownloadProgressModal />
        <DownloadModVersionSelectModal />
        <UpdateAllInstalledModsModal />
        <ConcerningPackageReviewModal/>
        <LaunchTypeModal v-if="canRenderLaunchTypeModal()" />

        <div class="router-view">
            <router-view name="subview" />
        </div>
    </div>
</template>

<script lang='ts' setup>
import { onUnmounted, ref } from 'vue';
import LocalFileImportModal from '../components/importing/LocalFileImportModal.vue';
import CategoryFilterModal from '../components/modals/CategoryFilterModal.vue';
import IncorrectGameDirectoryModal from '../components/modals/IncorrectGameDirectoryModal.vue';
import IncorrectSteamDirectoryModal from '../components/modals/IncorrectSteamDirectoryModal.vue';
import DependencyStringsModal from '../components/modals/DependencyStringsModal.vue';
import SteamInstallationValidationModal from '../components/modals/SteamInstallationValidationModal.vue';
import LaunchArgumentsModal from '../components/modals/LaunchArgumentsModal.vue';
import ProfileCodeExportModal from '../components/modals/ProfileCodeExportModal.vue';
import SortModal from '../components/modals/SortModal.vue';
import DownloadModVersionSelectModal from '../components/views/DownloadModVersionSelectModal.vue';
import DownloadProgressModal from '../components/views/DownloadProgressModal.vue';
import UpdateAllInstalledModsModal from '../components/views/UpdateAllInstalledModsModal.vue';
import { getStore } from '../providers/generic/store/StoreProvider';
import { State } from '../store';
import LaunchTypeModal from "../components/modals/launch-type/LaunchTypeModal.vue";
import appWindow from '../providers/node/app/app_window';
import ManagerActivityBar from '../components/navigation/ManagerActivityBar.vue';
import ProviderUtils from '../providers/generic/ProviderUtils';
import { Platform } from '../model/schema/ThunderstoreSchema';

const store = getStore<State>();

const { resetFilter } = useModFiltersComposable()

const importingLocalMod = ref<boolean>(false);

function canRenderLaunchTypeModal() {
    return ['linux', 'darwin'].includes(appWindow.getPlatform());
}

function closeSteamInstallationValidationModal() {
    isValidatingSteamInstallation.value = false;
}

async function validateSteamInstallation() {
    const res = await SteamInstallationValidator.validateInstallation(activeGame.value);
    if (res instanceof R2Error) {
        store.commit('error/handleError', res);
    } else {
        isValidatingSteamInstallation.value = true;
    }
}

function computeDefaultInstallDirectory(): string {
    switch(appWindow.getPlatform()){
        case 'win32':
            return path.resolve(
                process.env['ProgramFiles(x86)'] || process.env.PROGRAMFILES || 'C:\\Program Files (x86)',
                'Steam', 'steamapps', 'common', activeGame.value.steamFolderName
            );
        case 'linux':
            return path.resolve(os.homedir(), '.local', 'share', 'Steam', 'steamapps', 'common', activeGame.value.steamFolderName);
        case 'darwin':
            return path.resolve(os.homedir(), 'Library', 'Application Support', 'Steam',
                'steamapps', 'common', activeGame.value.steamFolderName);
        default:
            return '';
    }
}

function changeGameInstallDirectory() {
    const ror2Directory: string = settings.value.getContext().gameSpecific.gameDirectory || computeDefaultInstallDirectory();
    InteractionProvider.instance.selectFile({
        title: `Locate ${activeGame.value.displayName} Executable`,
        // Lazy reduce. Assume Linux name and Windows name are identical besides extension.
        // Should fix if needed, although unlikely.
        filters: (activeGame.value.exeName.map(value => {
            const nameSplit = value.split(".");
            return [{
                name: nameSplit[0],
                extensions: [nameSplit[1]]
            }]
        }).reduce((previousValue, currentValue) => {
            previousValue[0].extensions = [...previousValue[0].extensions, ...currentValue[0].extensions];
            return previousValue;
        })),
        defaultPath: ror2Directory,
        buttonLabel: 'Select Executable'
    }).then(async files => {
        if (files.length === 1) {
            try {
                const containsGameExecutable = activeGame.value.exeName.find(exeName => path.basename(files[0]).toLowerCase() === exeName.toLowerCase()) !== undefined
                if (containsGameExecutable) {
                    await settings.value.setGameDirectory(path.dirname(await FsProvider.instance.realpath(files[0])));
                } else {
                    showRor2IncorrectDirectoryModal.value = true;
                }
            } catch (e) {
                const err = R2Error.fromThrownValue(e, 'Failed to change the game folder');
                store.commit('error/handleError', err);
            }
        }
    });
}

function changeGameInstallDirectoryGamePass() {
    const ror2Directory: string = settings.value.getContext().gameSpecific.gameDirectory || computeDefaultInstallDirectory();
    InteractionProvider.instance.selectFile({
        title: `Locate gamelaunchhelper Executable`,
        filters: [{ name: "gamelaunchhelper", extensions: ["exe"] }],
        defaultPath: ror2Directory,
        buttonLabel: 'Select Executable'
    }).then(async files => {
        if (files.length === 1) {
            try {
                const containsGameExecutable = (path.basename(files[0]).toLowerCase() === "gamelaunchhelper.exe");
                if (containsGameExecutable) {
                    await settings.value.setGameDirectory(path.dirname(await FsProvider.instance.realpath(files[0])));
                } else {
                    throw new Error("The selected executable is not gamelaunchhelper.exe");
                }
            } catch (e) {
                const err = R2Error.fromThrownValue(e, 'Failed to change the game folder');
                store.commit('error/handleError', err);
            }
        }
    });
}

function computeDefaultSteamDirectory(): string {
    switch(appWindow.getPlatform()){
        case 'win32':
            return path.resolve(
                process.env['ProgramFiles(x86)'] || process.env.PROGRAMFILES || 'C:\\Program Files (x86)',
                'Steam'
            );
        case 'linux':
            return path.resolve(os.homedir(), '.local', 'share', 'Steam');
        case 'darwin':
            return path.resolve(os.homedir(), 'Library', 'Application Support', 'Steam');
        default:
            return '';
    }
}

async function checkIfSteamExecutableIsValid(file: string): Promise<boolean> {
    switch(appWindow.getPlatform()){
        case 'win32':
            return path.basename(file).toLowerCase() === "steam.exe"
        case 'linux':
            return path.basename(file).toLowerCase() === "steam.sh"
        case 'darwin':
            return path.basename(file).toLowerCase() === 'steam.app'
        default:
            return true;
    }
}

function changeSteamDirectory() {
    const steamDir: string = settings.value.getContext().global.steamDirectory || computeDefaultSteamDirectory();
    InteractionProvider.instance.selectFile({
        title: 'Locate Steam Executable',
        defaultPath: steamDir,
        filters: [{name: "steam", extensions: ["exe", "sh", "app"]}],
        buttonLabel: 'Select Executable'
    }).then(async files => {
        if (files.length === 1) {
            try {
                if (await checkIfSteamExecutableIsValid(files[0])) {
                    await settings.value.setSteamDirectory(path.dirname(await FsProvider.instance.realpath(files[0])));
                } else {
                    showSteamIncorrectDirectoryModal.value = true;
                }
            } catch (e) {
                const err = R2Error.fromThrownValue(e, 'Failed to change the Steam folder');
                store.commit('error/handleError', err);
            }
        }
    });
}

function setFunkyMode(value: boolean) {
    settings.value.setFunkyMode(value);
}

function browseDataFolder() {
    LinkProvider.instance.openLink('file://' + PathResolver.ROOT);
}

function browseProfileFolder() {
    LinkProvider.instance.openLink('file://' + profile.value.getProfilePath());
}

function toggleCardExpanded(expanded: boolean) {
    if (expanded) {
        settings.value.expandCards();
    } else {
        settings.value.collapseCards();
    }
    router.push({name: "manager.installed"});
}

async function toggleDarkTheme() {
    await settings.value.toggleDarkTheme();
    ThemeManager.apply();
}

function showLaunchParameters() {
    GameInstructions.getInstructionsForGame(activeGame.value, profile.value).then(instructions => {
        vanillaLaunchArgs.value = instructions.vanillaParameterList.map(value => `"${value}"`).join(' ');
    });

    GameRunnerProvider.instance.getGameArguments(activeGame.value, profile.value).then(target => {
        if (target instanceof R2Error) {
            doorstopTarget.value = "";
        } else {
            GameInstructionParser.parseList(target, activeGame.value, profile.value)
                .then(instructions => {
                    if (instructions instanceof R2Error) {
                        throw instructions;
                    }
                    doorstopTarget.value = instructions.map(value => `"${value}"`).join(' ');
                })
        }
    });

    launchParametersModel.value = settings.value.getContext().gameSpecific.launchParameters;
    showLaunchParameterModal.value = true;
}

function updateLaunchParameters() {
    settings.value.setLaunchParameters(launchParametersModel.value);
    showLaunchParameterModal.value = false;
}

async function copyLogToClipboard() {
    const fs = FsProvider.instance;
    let logOutputPath = "";
    switch (activeGame.value.packageLoader) {
        case PackageLoader.BEPINEX:
        case PackageLoader.BEPISLOADER:
            logOutputPath = path.join(profile.value.getProfilePath(), "BepInEx", "LogOutput.log");
            break;
        case PackageLoader.MELONLOADER:
            logOutputPath = path.join(profile.value.getProfilePath(), "MelonLoader", "Latest.log");
            break;
        case PackageLoader.RETURN_OF_MODDING:
            logOutputPath = path.join(profile.value.getProfilePath(), "ReturnOfModding", "LogOutput.log");
            break;
        case PackageLoader.GDWEAVE:
            logOutputPath = path.join(profile.value.getProfilePath(), "GDWeave", "GDWeave.log");
            break;
        case PackageLoader.UMM:
            logOutputPath = path.join(profile.value.getProfilePath(), "UMM", "Core", "Log.txt");
            break;
        case PackageLoader.RIVET:
            logOutputPath = path.join(profile.value.getProfilePath(), "Rivet", "RivetLoader.log");
            break;
    }
    const text = (await fs.readFile(logOutputPath)).toString();
    if (text.length >= 1992) {
        InteractionProvider.instance.copyToClipboard(text);
    } else {
        InteractionProvider.instance.copyToClipboard("```\n" + text + "\n```");
    }
}

async function copyTroubleshootingInfoToClipboard() {
    const content = await store.dispatch('profile/generateTroubleshootingString');
    InteractionProvider.instance.copyToClipboard('```' + content + '```');
}

async function changeDataFolder() {
    try {
        const folder = await DataFolderProvider.instance.showSelectionDialog();

        if (folder === null) {
            return;
        }

        await DataFolderProvider.instance.throwForInvalidFolder(folder);
        await DataFolderProvider.instance.writeOverrideFile(folder);
        await settings.value.setDataDirectory(folder);
        InteractionProvider.instance.restartApp();
    } catch(err) {
        store.commit("error/handleError", R2Error.fromThrownValue(err));
        return
    }
}

async function handleSettingsCallbacks(invokedSetting: any) {
    switch(invokedSetting) {
        case "BrowseDataFolder":
            browseDataFolder();
            break;
        case "BrowseProfileFolder":
            browseProfileFolder();
            break;
        case "ChangeGameDirectory":
            changeGameInstallDirectory();
            break;
        case "ChangeGameDirectoryGamePass":
            changeGameInstallDirectoryGamePass();
            break;
        case "ChangeSteamDirectory":
            changeSteamDirectory();
            break;
        case "CopyLogToClipboard":
            copyLogToClipboard();
            break;
        case "CopyTroubleshootingInfoToClipboard":
            copyTroubleshootingInfoToClipboard();
            break;
        case "ToggleDownloadCache":
            await store.dispatch('download/toggleIgnoreCache');
            break;
        case "ValidateSteamInstallation":
            validateSteamInstallation();
            break;
        case "SetLaunchParameters":
            showLaunchParameters();
            break;
        case "ChangeProfile":
            router.push({name: "profiles"});
            break;
        case "ImportLocalMod":
            importingLocalMod.value = true;
            break;
        case "ToggleFunkyMode":
            setFunkyMode(!settings.value.getContext().global.funkyModeEnabled);
            break;
        case "SwitchTheme":
            toggleDarkTheme();
            document.documentElement.classList.toggle('html--dark', settings.value.getContext().global.darkTheme);
            break;
        case "SwitchCard":
            toggleCardExpanded(!settings.value.getContext().global.expandedCards);
            break;
        case "EnableAll":
            await store.dispatch(
                "profile/enableModsOnActiveProfile",
                {mods: localModList.value}
            );
            await router.push({name: "manager.installed"});
            break;
        case "DisableAll":
            await store.dispatch(
                "profile/disableModsFromActiveProfile",
                {mods: localModList.value}
            );
            await router.push({name: "manager.installed"});
            break;
        case "UpdateAllMods":
            store.commit("openUpdateAllModsModal");
            break;
        case "ShowDependencyStrings":
            showDependencyStrings.value = true;
            break;
        case "ChangeDataFolder":
            await changeDataFolder();
            break;
        case "CleanCache":
            CacheUtil.clean();
            break;
        case "ToggleIgnoreStore":
            await settings.value.setIgnoreStore(!settings.value.getIgnoreStore());
            await ProviderUtils.setupGameProviders(activeGame.value, activeGame.value.activePlatform.storePlatform);
            break;
    }
}

store.dispatch('profile/loadOrderingSettings');
store.commit('modFilters/reset');

onUnmounted(() => {
    resetFilter();
});
</script>

<style lang="scss">

.manager-main-view {
    display: flex;
    flex: 1;
    width: 100%;
}

.router-view {
    display: flex;
    flex: 1;
    width: 100%;
}

.game-icon {
    height: 1.125rem;
    border-radius: 2px;
}

.vertical-break {
    height: 1.25rem;
    width: 1px;
    margin: 0 0.25rem;
    background-color: var(--border, #e1e1e1);
    border-radius: 5px;
    align-self: center;
    flex-shrink: 0;
}
</style>
